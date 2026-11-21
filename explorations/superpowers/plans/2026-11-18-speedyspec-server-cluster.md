# SpeedySpec Server Cluster Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or
> superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Managed mode runs on Kubernetes with several replicas sharing one `data_dir` volume. One elected **leader**
pod runs every release operation; the others are **followers** that serve whatever the leader makes live:
- apps switch on every pod together, in two steps (staged, then active);
- a pod that cannot prepare a release goes not-ready instead of serving a mixed set;
- the server upgrades pod by pod, by plan 3's socket handoff, each pod finishing its trial before the next starts;
- a failed trial on any pod stops the rollout and reverts the pods already upgraded.

A single host (`leadership = "single"`) runs the same code as a cluster of one, and behaves exactly as it does after
plan 3.

**Architecture:**
- **Two new contracts in `-domain`**, each bound separately:
  - `Leadership`: whether this pod leads now, its Lease epoch, and who the leader is;
  - `ClusterMembers`: write this pod's member file, read every pod's.
- **Implementations:**
  - `single` (always leader, epoch 0) and the in-memory pair for tests, in `-infra-memory`;
  - member files on the shared volume, and a lease-file leadership for multi-process tests on one host, in
    `-infra-fs`;
  - a `coordination.k8s.io/v1` Lease, in the new crate `-infra-kubernetes`.
- **`-app`** gains:
  - leader-gated writes: every `state.json` write carries the leader's epoch, and a manager that is not the leader
    refuses to write;
  - the two-step switch, the follower loop, member-aware pruning, picking up after a leader change;
  - the rollout state machine for server upgrades.
- **The binary:**
  - `[cluster]` config;
  - a heartbeat loop, a follow loop and a leader loop in the active generation;
  - a probe listener answering `/readyz` and `/livez`;
  - per-pod server state in the member file;
  - generations asking their keeper for the upgrade when it is their pod's turn.
- **Who leads is the pod.** The Lease holder is the pod's name, and the pod's active generation renews it. After a
  handoff the new generation renews under the same name, so leadership does not move.

**Tech Stack:** Plan 3's stack, plus `kube` 5 (`client`, `rustls-tls`, `ring`; no default features) and `k8s-openapi`
0.29 (`latest`), as `packages/speedyspec-cli/crates/speedyspec-driver-kubernetes` uses them. `kube` runs on its own
current-thread runtime inside `-infra-kubernetes`, so the domain contract stays synchronous.

**Spec:** `docs/superpowers/specs/2026-11-17-speedyspec-server-core-design.md`, especially "Managed mode on
Kubernetes: one pod leads", "Who runs release operations", "On disk", "Packages", "Socket handoff" and "Testing"
(the "Leader and followers" and "Kubernetes" bullets).

**This is plan 4 of 4.** Plan 3 is [SpeedySpec/speedyspec-server#2](https://github.com/SpeedySpec/speedyspec-server/pull/2),
merged: branch `feat/cluster` from `main`.

## Global Constraints

- Work in `packages/speedyspec-server/`. Do not push or open a PR without Jacob's go-ahead.
- Rust 1.99, edition 2024, default rustfmt. `cargo fmt --all` before every commit. Conventional Commits, each ending
  with `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.
- Contracts live in `-domain`; each implementation lives in its own crate and can be replaced on its own. Salvo
  appears only in `-presentation` and the binary. `kube` appears only in `-infra-kubernetes`.
- TLS stays on `ring`: `cargo tree -i aws-lc-sys --target all` prints nothing (CI checks this).
- **Only the leader writes shared state.** Followers write nothing under `data_dir` except their own
  `cluster/members/<pod>.json`. Within one pod, `<run_dir>/state.lock` (pod-local) still serialises writes. `flock`
  is never taken on the shared volume.
- **Every shared write carries an epoch.** A follower ignores any `state.json` whose epoch is older than one it has
  already seen.
- `leadership = "single"` must leave every plan 1–3 test passing unchanged.
- Immutable mode is untouched: a `[cluster]` table in immutable mode is a config error.
- Unix only. Every test passes on macOS arm64 and on Linux musl (plan 3's Docker command). Tests run with
  `cargo test --workspace --features speedyspec-server/test-hooks`. The Kubernetes end-to-end test is `#[ignore]`d
  and runs only in its own CI job.
- Every refusal names what was refused and why. A follower refuses a mutating command with the leader's pod name.
  Logs are JSON lines, and every cluster step is its own event: `leader_acquired`, `leader_lost`, `staged`,
  `prepared`, `activated`, `switch_abandoned`, `member_stale`, `rollout_step`, `rollout_failed`, `rollout_reverted`,
  `stale_epoch_ignored`.

Spec values used across tasks (the `[cluster]` keys are my calls):

| Value | Key | Default |
|---|---|---|
| Leadership | `cluster.leadership` | `"single"` (`"kubernetes"`; `"file"` only under `test-hooks`) |
| This pod's name | `cluster.pod` | `$POD_NAME`, else the host name |
| Lease name / namespace | `cluster.lease_name` / `cluster.lease_namespace` | `speedyspec-server` / the service account's namespace file |
| Lease duration | `cluster.lease_duration` | 15 s |
| Renew deadline | `cluster.renew_deadline` | 10 s (must be below the lease duration) |
| Retry period | `cluster.retry_period` | 2 s (must be below the renew deadline) |
| Heartbeat interval | fixed | 2 s |
| A member counts as stale after | fixed | 10 s without a new heartbeat |
| `state.json` poll interval | `cluster.poll_interval` | 2 s |
| Prepare timeout | `cluster.prepare_timeout` | 60 s |
| Probe listener | `cluster.probe` | unset (no probe listener) |

## Review Focus

Each line has a test in the task named in brackets.

1. **Clock skew between pods.** Staleness is judged by the reader's own monotonic clock: a member counts as stale
   when its heartbeat *counter* has not changed for 10 s as this pod saw it. The writer's wall clock is never compared
   to the reader's. A pod whose clock is an hour off must count as live while it heartbeats, and as stale once it
   stops. [Task 4]
2. **A torn read on a network file system.** A reader that finds a `state.json` or member file it cannot parse keeps
   its last good copy and retries at the next poll. It does not log an error every poll, does not treat the member as
   stale on that alone, and never acts on a partial file. [Task 2]
3. **A leader that loses its Lease mid-switch.** Once its renew deadline passes, it stops writing at once, even mid-
   operation: the write that would follow is refused with `not the leader`. A late write that still lands (an older
   epoch) is ignored by every follower, and the new leader finishes or abandons the staged switch. [Tasks 3 and 5]
4. **A pod killed during a two-step switch.** The leader stops waiting for that pod as soon as its heartbeat goes
   stale. It does not hold the switch for the whole `prepare_timeout`, and it never counts a stale pod's old
   `prepared` report. [Task 4]
5. **Pruning under a lagging follower.** A release that any live member reports as serving or prepared is never
   removed, even when it is beyond `keep` and no longer active in `state.json`. [Task 4]

## File Structure

```
crates/speedyspec-server-domain/src/cluster.rs     Leadership, ClusterMembers, LeaderView, MemberReport, PodServer
crates/speedyspec-server-domain/src/state.rs       + epoch, staged: Option<StagedSet>, rollout: Option<Rollout>
crates/speedyspec-server-infra-memory/src/cluster.rs  SingleLeadership, MemoryLeadership, MemoryMembers
crates/speedyspec-server-infra-fs/src/members.rs   FsClusterMembers (cluster/members/<pod>.json)
crates/speedyspec-server-infra-fs/src/lease.rs     FileLeaseLeadership (one host, tests only)
crates/speedyspec-server-infra-kubernetes/         new crate: KubeLeadership (Lease)   tests/lease.rs
crates/speedyspec-server-app/src/cluster.rs        Cluster (leader gate, liveness), follow(), leader_tick()
crates/speedyspec-server-app/src/rollout.rs        the rollout state machine
crates/speedyspec-server-app/src/releases.rs       writes gated and stamped; switch staged; prune member-aware
crates/speedyspec-server/src/config.rs             [cluster]
crates/speedyspec-server/src/cluster.rs            wiring: loops, probe handler, per-pod server state
crates/speedyspec-server/src/{generation,keeper,control,managed}.rs   rollout turn, member-file confirm, refusals
crates/speedyspec-server/tests/cluster.rs          three processes on one data_dir (file leadership)
crates/speedyspec-server/tests/kubernetes_e2e.rs   #[ignore]: k3s, three replicas
deploy/kubernetes/                                 ServiceAccount, Role, RoleBinding, PVC, Deployment, Service
scripts/k3s.sh, Dockerfile                         the throwaway cluster and the musl image
docs/cluster.md, README.md                         how it works; how to run it
```

---

### Task 1: Domain contracts and the release state's cluster fields

**Files:**
- Create: `crates/speedyspec-server-domain/src/cluster.rs`, `crates/speedyspec-server-infra-memory/src/cluster.rs`
- Modify: `crates/speedyspec-server-domain/src/{lib,state}.rs`, `crates/speedyspec-server-infra-memory/src/lib.rs`
- Test: in-crate tests in both files, and `state.rs`'s tests

**Interfaces:**
- Produces, in `-domain`:
  - `pub struct LeaderView { pub is_leader: bool, pub epoch: u64, pub leader: Option<String> }`.
  - `pub trait Leadership: Send + Sync`:
    - `fn view(&self) -> LeaderView`. It must reflect the renew deadline at the moment of the call: once the deadline
      has passed without a renewal, `is_leader` is false even if no background step has run yet.
    - `fn changed(&self) -> std::sync::mpsc::Receiver<LeaderView>`: every change, for logging and waking the loops.
    - `fn step_down(&self)`: releases the Lease when one is held (on `SIGTERM`), so a successor does not wait out
      the lease duration.
  - `pub struct MemberReport` (serde, every field defaulted):
    - `pod: String`;
    - `heartbeat: u64`, a counter that increases on every write;
    - `serving: u64`, the `state.json` revision whose active set this pod serves;
    - `prepared: Option<u64>`, the staged revision it has prepared;
    - `prepare_failed: Option<(u64, String)>`;
    - `holds: Vec<(AppId, Version)>`, every release it serves or has prepared;
    - `ready: bool`;
    - `server: PodServer`.
  - `pub struct PodServer { pub running: Option<Version>, pub confirmed: Option<Version>, pub trial: Option<Version>,
    pub failed: BTreeMap<Version, String> }`: this pod's own server binary state in a cluster.
  - `pub trait ClusterMembers: Send + Sync { fn write(&self, report: &MemberReport) -> Result<(), StateError>;
    fn read_all(&self) -> Result<Vec<MemberReport>, StateError>; }`.
- In `ReleaseState` (each `#[serde(default)]`, so plan 3 state files still load):
  - `epoch: u64`: the epoch of the leader that wrote this.
  - `staged: Option<StagedSet>`, where `StagedSet { revision: u64, targets: Vec<(AppId, Version)>, since_ms: u64 }`
    and `revision` is the revision the set was staged at.
  - `rollout: Option<Rollout>`, where `pub struct Rollout { pub target: Version, pub previous: Version, pub group:
    Option<PendingGroup>, pub order: Vec<String>, pub done: Vec<String>, pub turn: Option<String>, pub reverting:
    bool, pub failure: Option<(String, String)> }`. The failure records the pod and the reason; Task 6 gives the
    fields their meaning.
- In `-infra-memory`:
  - `SingleLeadership::new(pod)`: always leader, epoch 0.
  - `MemoryLeadership`, shared by several managers in one process: `elect(pod)` makes `pod` the leader and increments
    the epoch; `expire()` makes nobody the leader.
  - `MemoryMembers`, shared: `fail_next_read()`, and `tamper(pod, |report| …)` for tests.

- [ ] **Step 1: Write the failing tests.**
  - `cluster_fields_default_when_absent` (a plan 3 `state.json` loads with `epoch == 0`, `staged == None` and
    `rollout == None`). Replace the existing test that uses `"epoch":7` as its example of an unknown field with
    another unknown name.
  - `staged_set_round_trips`.
  - `member_report_round_trips_and_tolerates_missing_fields`.
  - `single_leadership_always_leads`.
  - `memory_leadership_elect_bumps_epoch_and_notifies`: two `changed()` receivers see the same views in order.
  - `memory_leadership_expire_leaves_no_leader`.
- [ ] **Step 2: Run** `cargo test -p speedyspec-server-domain -p speedyspec-server-infra-memory` → FAIL (types
  missing).
- [ ] **Step 3: Implement** the types, the traits and the in-memory implementations.
- [ ] **Step 4: Run** the same command → PASS, and the whole suite with the feature → PASS.
- [ ] **Step 5: Commit.** `feat(domain): leadership and cluster member contracts, and the state's cluster fields`

---

### Task 2: Member files and the lease file on disk

**Files:**
- Create: `crates/speedyspec-server-infra-fs/src/{members,lease}.rs`
- Modify: `crates/speedyspec-server-infra-fs/src/lib.rs`
- Test: `crates/speedyspec-server-infra-fs/tests/cluster.rs`

**Interfaces:**
- Consumes: Task 1's `MemberReport`, `ClusterMembers`, `Leadership` and `LeaderView`.
- Produces:
  - `FsClusterMembers::new(data_dir, pod)`:
    - `write` replaces `<data_dir>/cluster/members/<pod>.json` atomically: write a temp file, fsync, rename, then
      fsync the directory. There is no lock.
    - `read_all` returns every member that parses. If a file does not parse, the reader returns the last good copy
      it read of that file, if it has one, and logs `member_unreadable` once per run of failures, not once per read
      (Review Focus 2).
  - `FileLeaseLeadership::start(path, pod, durations)`: one lease file holding holder, renewed-at (ms since the Unix
    epoch) and the transition count.
    - It renews on a thread. Taking the lease is a rename-into-place.
    - The previous holder must have missed its lease duration. On one host all processes share one clock, so wall
      time is safe here.
    - It exists only for the multi-process tests on one host: the binary accepts `leadership = "file"` only when built
      with `test-hooks` (Task 7).

- [ ] **Step 1: Write the failing tests.**
  - `member_write_is_atomic_and_read_all_lists_every_pod`.
  - `torn_member_file_keeps_last_good_copy` (Review Focus 2): write a good file, overwrite it with half its bytes →
    `read_all` still returns the good report, and logs `member_unreadable` once over three reads.
  - `file_lease_one_holder_at_a_time`: two leaderships on one file → exactly one leads.
  - `file_lease_passes_on_after_holder_stops_renewing`: within lease duration plus retry period; the epoch increases
    by one.
  - `file_lease_view_turns_false_at_renew_deadline_without_a_renewal`: pause the renew thread with a test switch;
    `view().is_leader` turns false by the renew deadline, before anyone else can take the lease.
  - `step_down_releases_at_once`.
- [ ] **Step 2: Run** `cargo test -p speedyspec-server-infra-fs --test cluster` → FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** → PASS.
- [ ] **Step 5: Commit.** `feat(infra-fs): member files and a one-host lease file`

---

### Task 3: Leader-gated, epoch-stamped writes

**Files:**
- Create: `crates/speedyspec-server-app/src/cluster.rs`
- Modify: `crates/speedyspec-server-app/src/{lib,releases}.rs`, `crates/speedyspec-server/src/managed.rs`
- Test: `crates/speedyspec-server-app/tests/cluster.rs` (three managers in one process, sharing a
  `MemoryStateStore`, `MemoryMembers`, `MemoryLeadership` and one `MemoryReleaseStore` directory)

**Interfaces:**
- Consumes: Task 1.
- Produces:
  - `pub struct Cluster { pod: String, leadership: Arc<dyn Leadership>, members: Arc<dyn ClusterMembers>,
    poll_interval: Duration, prepare_timeout: Duration }`, and `Cluster::single(pod)`. The single cluster uses
    `SingleLeadership` and a member store that keeps one report in memory, so a single host writes no member file.
  - `ReleaseManager::new(…)` takes an `Arc<Cluster>` as its last argument. `managed::start` passes
    `Cluster::single`, until Task 7 wires `[cluster]`.
  - `commit_state` checks `leadership.view()` immediately before saving:
    - not the leader → `ReleaseError::NotLeader { leader: Option<String> }`, which reads "this pod is not the
      leader; run release commands on <leader>" or, with no leader, "there is no leader right now; try again
      shortly";
    - otherwise `next.epoch = view.epoch`, then save.
  - `ReleaseManager::refresh(&self) -> Result<bool, ReleaseError>`: reloads `state.json` into memory, unless its
    epoch is older than the newest epoch seen, which is ignored with `stale_epoch_ignored`. Returns whether the state
    changed.
  - Every mutating method (`install`, `switch`, `rollback`, `pin`, `unpin`, `update`, `begin_server_switch`) refuses
    up front with `NotLeader` on a follower, before downloading anything.

- [ ] **Step 1: Write the failing tests.**
  - `follower_refuses_mutations_naming_the_leader`.
  - `writes_carry_the_leader_epoch`.
  - `leader_past_renew_deadline_refuses_the_next_write` (Review Focus 3): `MemoryLeadership::expire()` between a
    switch's prepare and its save → `NotLeader`, and `state.json` unchanged.
  - `older_epoch_state_is_ignored_by_refresh`: elect A (epoch 1), then B (epoch 2), and B writes. A late write
    stamped epoch 1, written straight into the store, is ignored by C's `refresh`.
  - Every existing `-app` test passes with `Cluster::single`.
- [ ] **Step 2: Run** `cargo test -p speedyspec-server-app` → FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** the whole suite with the feature → PASS (single mode unchanged).
- [ ] **Step 5: Commit.** `feat(app): only the leader writes state, and every write carries its epoch`

---

### Task 4: The two-step switch, the follower loop and member-aware pruning

**Files:**
- Modify: `crates/speedyspec-server-app/src/{cluster,releases}.rs`
- Test: `crates/speedyspec-server-app/tests/cluster.rs`

**Interfaces:**
- Consumes: Task 3.
- Produces:
  - `Liveness::observe(&mut self, reports: &[MemberReport], now: Instant) -> Vec<String>` returns the live pods.
    `Liveness` is owned by each manager's `Cluster`.
    - A pod is live while its `heartbeat` counter has changed within the last 10 s of *this reader's* clock (Review
      Focus 1).
    - A pod seen for the first time counts as live.
  - **The leader's `switch`** (plain switches only: a set that includes the server is Task 6's rollout):
    1. Check the set and write `staged = Some(StagedSet { revision, targets, since_ms })`, leaving `active`
       unchanged; log `staged`.
    2. Prepare the set locally, through the runtimes, holding each `Prepared` without activating it. Report
       `prepared = Some(revision)`, or `prepare_failed`, and log `prepared`.
    3. Wait, polling members every `poll_interval`, until every live pod has reported on this revision, or
       `prepare_timeout` passes:
       - a pod stops counting the moment it goes stale (Review Focus 4);
       - a pod that went stale and then came back must report again;
       - a report made before the staged revision never counts.
    4. If at least one pod prepared, write the targets as `active`, clear `staged`, activate locally, and log
       `activated` with the prepared and failed pods. Otherwise clear `staged`, mark the versions failed with every
       pod's reason, log `switch_abandoned`, and return `ReleaseError::Prepare`.
    5. With `Cluster::single` there is one member and no waiting, so `release switch` returns as quickly as before.
  - **`ReleaseManager::follow(&self) -> FollowOutcome`**, one step on a follower, every `poll_interval`:
    1. `refresh`.
    2. If a set is staged and this pod has not reported on its revision, prepare it and report.
    3. If the active set differs from what this pod serves, activate it, using the held `Prepared` when there is one
       and preparing now otherwise. If it cannot prepare, keep serving the old set and report `ready = false` (the
       probe answers 503) until a later active set prepares.
    4. Report `serving`, `holds` and `ready` every step.
    - `FollowOutcome` holds `{ changed: bool, ready: bool }`.
  - **Prune:** never remove a version that any live member's `holds` lists (Review Focus 5).

- [ ] **Step 1: Write the failing tests** (three managers A, B and C, with A elected). Time is the `Instant` passed to
  `Liveness::observe`, through a clock the `Cluster` takes, so the tests advance it without sleeping:
  - `two_step_switch_lands_on_all_three`: after A's `switch`, one `follow` step on B and C → all three runtimes
    serve 2.0.0.
  - `member_failing_prepare_goes_not_ready_and_others_switch`: `fail_prepare` on C → A and B switch; C keeps 1.0.0
    with `ready == false`; and C turns ready once a later set prepares.
  - `no_pod_prepares_abandons_and_marks_failed`.
  - `stale_member_is_not_waited_for` (Review Focus 4): C stops heartbeating mid-switch → A activates within the
    staleness window plus one poll, well before `prepare_timeout`.
  - `report_from_before_the_staged_revision_does_not_count`.
  - `prune_keeps_releases_live_members_hold` (Review Focus 5): `keep = 1`, and B still holds 1.0.0 → 1.0.0 is kept
    until B reports 2.0.0.
  - `liveness_ignores_writer_clock` (Review Focus 1): a report whose fields claim a wall time an hour in the future
    or past is live while its counter moves, and stale 10 s after it stops.
- [ ] **Step 2: Run** `cargo test -p speedyspec-server-app --test cluster` → FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** the whole suite with the feature → PASS.
- [ ] **Step 5: Commit.** `feat(app): two-step switches across pods, the follower loop and member-aware pruning`

---

### Task 5: Changing leader

**Files:**
- Modify: `crates/speedyspec-server-app/src/cluster.rs`, `src/releases.rs`
- Test: `crates/speedyspec-server-app/tests/cluster.rs`

**Interfaces:**
- Consumes: Tasks 3 and 4.
- Produces: `ReleaseManager::take_over(&self) -> Result<TakeOver, ReleaseError>`, called once when a manager's
  `Leadership` turns it into the leader. It:
  1. deletes stale `.partial` directories (`clean_partials`);
  2. refreshes, and reads every member file;
  3. finishes a staged switch. It re-checks every target's manifest in the store: if one is missing or
     unreadable, it abandons the switch (`switch_abandoned`, reason "the new leader could not verify <app>
     <version>"); otherwise it carries on from step 3 of Task 4, counting reports already made on that revision;
  4. carries on a rollout from where `state.rollout` shows it (Task 6).
  - `TakeOver` holds `{ finished: Option<u64>, abandoned: Option<(u64, String)> }`.

- [ ] **Step 1: Write the failing tests.**
  - `leader_killed_mid_switch_new_leader_finishes`: A stages and is expired before activating; B is elected →
    B's `take_over` activates the set, and C follows.
  - `new_leader_abandons_unverifiable_switch`: delete the staged release's directory before B takes over.
  - `deposed_leader_late_write_is_ignored` (Review Focus 3, end to end): A is expired mid-switch, B is elected and
    writes, then A's late save, stamped with the older epoch, lands in the store → B and C ignore it, and the
    active set is B's.
  - `take_over_cleans_partials`.
- [ ] **Step 2: Run** → FAIL. **Step 3: Implement.** **Step 4: Run** the whole suite → PASS.
- [ ] **Step 5: Commit.** `feat(app): a new leader finishes or abandons the switch it inherits`

---

### Task 6: The rolling server upgrade, as a state machine

**Files:**
- Create: `crates/speedyspec-server-app/src/rollout.rs`
- Modify: `crates/speedyspec-server-app/src/releases.rs`
- Test: in-crate tests in `rollout.rs`, plus `tests/cluster.rs`

**Interfaces:**
- Consumes: Tasks 3–5, Task 1's `Rollout`, and plan 3's `PendingGroup`.
- Produces:
  - `pub fn next_step(rollout: &Rollout, live: &[String], members: &[MemberReport]) -> RolloutStep`, a pure
    function. `RolloutStep` is one of:
    - `Turn(pod)`: it is this pod's turn to upgrade;
    - `Wait`;
    - `Done`: every live pod confirmed the target;
    - `Failed { pod, reason }`;
    - `Reverted`: every pod in `done` is back on `previous`.
  - The rules of `next_step`:
    - `order` is the live pods sorted by name, with the leader last (my call: the leader keeps coordinating until
      the end).
    - A pod finishes its turn when its `PodServer` shows `confirmed == target` and no trial.
    - A pod whose `failed` holds the target has failed. The rollout then reverts: every pod in `done` gets a turn
      back to `previous`, one at a time, in reverse order.
    - A pod that goes stale during its turn is skipped. It is not counted as done, and pods that join later upgrade
      at the end.
  - `ReleaseManager::begin_server_switch`:
    - with `Cluster::single`, it keeps plan 3's behaviour exactly;
    - in a cluster, the leader writes `rollout` (with the group, if any) and `server.trial = target`, and replies
      `{"rollout": …}` instead of `{"upgrade": …}`. The pods then upgrade through Task 8's wiring.
  - `ReleaseManager::advance_rollout(&self) -> Result<Option<RolloutStep>, ReleaseError>`, the leader's step:
    - it applies `next_step`, writing `turn`, `done`, `reverting` and `failure`;
    - on `Done`: `server.confirmed = target`, then clear `trial`, `rollout` and the group; the group's apps become
      `active`;
    - on `Reverted`: the target joins `server.failed` with the failing pod's reason, and the group's apps stay on
      their previous versions;
    - it logs `rollout_step`, `rollout_failed` and `rollout_reverted`.

- [ ] **Step 1: Write the failing tests.**
  - The `next_step` unit tests: `turns_in_name_order_leader_last`, `waits_during_a_trial`, `done_when_all_confirmed`,
    `failed_trial_starts_revert_in_reverse_order`, `stale_pod_skipped`, `late_joiner_upgrades_last`.
  - `single_mode_server_switch_unchanged` (the existing plan 3 tests in `-app` still pass).
  - `rollout_failure_marks_version_failed_and_keeps_group_apps`.
- [ ] **Step 2: Run** → FAIL. **Step 3: Implement.** **Step 4: Run** the whole suite → PASS.
- [ ] **Step 5: Commit.** `feat(app): pod-by-pod server rollout with revert on a failed trial`

---

### Task 7: `[cluster]` config and the loops in the generation

**Files:**
- Create: `crates/speedyspec-server/src/cluster.rs`
- Modify: `crates/speedyspec-server/src/{config,generation,managed,control,keeper}.rs`,
  `crates/speedyspec-server-handoff/src/protocol.rs`
- Test: `crates/speedyspec-server/src/config.rs` tests, `crates/speedyspec-server/tests/control.rs`

**Interfaces:**
- Consumes: Tasks 1–6.
- Produces:
  - **`Config.cluster: Option<ClusterConfig>`**, with the keys and defaults in the table above. It is refused:
    - in immutable mode;
    - when `renew_deadline >= lease_duration`, or `retry_period >= renew_deadline`;
    - for `leadership = "file"` unless built with `test-hooks`.
  - **The active generation, in managed mode:**
    - builds `Cluster` from config: `single` → `SingleLeadership`; `file` → `FileLeaseLeadership` at
      `<data_dir>/cluster/lease`; `kubernetes` → Task 9's `KubeLeadership`. `FsClusterMembers` is used whenever
      `leadership` is not `single`.
    - runs, while not draining:
      - a heartbeat every 2 s;
      - as leader: `advance_rollout`, pruning, and auto-update; and once, on becoming leader, `take_over`;
      - as follower: `follow` every `poll_interval`.
    - logs `leader_acquired` and `leader_lost` from `changed()`.
    - on a draining generation: stops all three loops and stops renewing the Lease. The successor generation of the
      same pod renews under the same pod name.
  - **The probe listener**, `cluster.probe`:
    - the keeper binds it like any listener. `SocketInfo` gains `probe: bool` (serde default `false`; protocol 1 is
      unchanged in meaning), and the generation serves only `/readyz` and `/livez` on it;
    - `/livez` answers 200 while the generation runs;
    - `/readyz` answers 200 once the generation is ready and its last `follow` reported `ready`, and 503 otherwise
      (with the reason in the body).
  - **`status`** gains a `cluster` line, `cluster<TAB><pod><TAB>leader|follower<TAB><leader or ->`, and one `member`
    line per member: `member<TAB><pod><TAB>live|stale<TAB><serving revision><TAB>ready|not-ready<TAB><server
    version>`. `cluster.leadership = "single"` prints no new lines, so plan 1–3 `status` tests are unchanged.
  - **Followers** refuse `install`, `switch`, `rollback`, `pin`, `unpin` and `reload` with Task 3's message. Reads
    work on any pod.

- [ ] **Step 1: Write the failing tests.**
  - Config: `cluster_defaults`, `cluster_refused_in_immutable_mode`, `renew_deadline_must_be_below_lease_duration`,
    `file_leadership_needs_test_hooks` (a `#[cfg(not(feature = "test-hooks"))]` test).
  - `tests/control.rs`: `single_status_has_no_cluster_lines`.
  - `probe_answers_livez_and_readyz` (one process, `leadership = "file"`).
- [ ] **Step 2: Run** → FAIL. **Step 3: Implement.** **Step 4: Run** the whole suite → PASS.
- [ ] **Step 5: Commit.** `feat: [cluster] config, the leader and follower loops, and readiness probes`

---

### Task 8: Rollout turns in the generation and the keeper

**Files:**
- Modify: `crates/speedyspec-server/src/{cluster,generation,keeper,managed,control}.rs`,
  `crates/speedyspec-server-app/src/releases.rs`
- Test: `crates/speedyspec-server/tests/cluster.rs`

**Interfaces:**
- Consumes: Tasks 6 and 7, plan 3's keeper upgrade path and `ToKeeper::Upgrade` (unused until now).
- Produces:
  - **The turn.** When `state.rollout.turn` names this pod and its running server version is not the turn's target
    (`target`, or `previous` when reverting), the active generation sends its keeper
    `ToKeeper::Upgrade { version, binary }`. The keeper runs the same path as `KeeperCommand::Upgrade` (start N+1,
    drain, trial, confirm or fall back) and logs that the request came from the rollout. Each new generation applies
    the rollout's group to its own runtimes in memory, without writing `state.json`: `apply_pending_group` gains a
    local-only form for clusters.
  - **Per-pod server state** in a cluster lives in the member file's `PodServer`, not in `state.json`:
    - the trial confirm (`ConfirmServer`) and a failed trial (`Resume { failed }`) update `PodServer`;
    - the leader alone moves `state.json.server` (Task 6's `Done` or `Reverted`);
    - `managed::confirmed_binary` reads the pod's `PodServer.confirmed`, and falls back to
      `state.json.server.confirmed`;
    - a follower pod never registers its running binary into the shared store (that is a shared write). On first
      start it runs `state.server.confirmed`'s kept binary when there is one, and its own executable otherwise;
    - with `leadership = "single"`, all of this stays exactly as in plan 3.
  - **Keeper re-exec** (plan 3) works unchanged per pod. The leader's own turn is last, and its handoff keeps
    the Lease (same pod name).

- [ ] **Step 1: Write the failing tests** in `tests/cluster.rs`, with three keepers (pods `a`, `b`, `c`) on one
  `data_dir`, each with its own `run_dir` and port, `leadership = "file"`, and plan 3's `SPEEDYSPEC_TEST_*` hooks.
  This task's tests:
  - `rolling_upgrade_goes_pod_by_pod`: `release switch server@0.2.0` on the leader → the pods upgrade one at a time
    (never two on trial at once, which a monitor thread checks through `status`), the leader last; `state.json`
    ends with `server.confirmed = 0.2.0`; and `GET /` never fails on any pod.
  - `rolling_upgrade_stops_and_reverts_on_one_failed_trial`: `SPEEDYSPEC_TEST_EXIT_VERSION=0.2.0` on pod `b` only →
    pod `a` (already upgraded) goes back to 0.1.0, `c` never upgrades, and `server.failed` holds 0.2.0 with `b`'s
    reason.
  - `follower_first_start_does_not_write_shared_store`.
- [ ] **Step 2: Run** `cargo test -p speedyspec-server --features test-hooks --test cluster` → FAIL.
- [ ] **Step 3: Implement.** **Step 4: Run** the whole suite → PASS on macOS and Linux musl.
- [ ] **Step 5: Commit.** `feat: server upgrades roll pod by pod through each pod's keeper`

---

### Task 9: `-infra-kubernetes`: the Lease

**Files:**
- Create: `crates/speedyspec-server-infra-kubernetes/{Cargo.toml,src/lib.rs,tests/lease.rs,tests/support/fake_api.rs}`
- Modify: `crates/speedyspec-server/{Cargo.toml,src/cluster.rs}`

**Interfaces:**
- Consumes: Task 1's `Leadership`.
- Produces: `KubeLeadership::start(client_config, LeaseSettings { name, namespace, pod, lease_duration,
  renew_deadline, retry_period }) -> Result<KubeLeadership, String>`, a `Leadership`.
  - In-cluster config comes from the service account (`kube::Config::incluster`). Tests pass a config pointing at
    the fake API.
  - The `coordination.k8s.io/v1` Lease is read with `get`, made with `create` when missing, and taken or renewed
    with `update` (optimistic, `resourceVersion`):
    - `holderIdentity` = pod;
    - `leaseDurationSeconds`;
    - `renewTime`, `acquireTime`;
    - `leaseTransitions` is the epoch, and increases on every change of holder.
  - Another holder's lease can be taken only once `renewTime + leaseDurationSeconds` has passed *by this pod's
    observation*. The pod notes the `renewTime` value and when, on its own monotonic clock, it last saw that value
    change. This is the same rule as Review Focus 1, and it avoids comparing clocks.
  - `view()` turns `is_leader` false the moment `renew_deadline` has passed since the last successful renewal,
    without waiting for the loop.
  - `step_down` clears `holderIdentity`, using `update`.
  - Runs on its own thread with a current-thread tokio runtime; nothing async leaks out.
  - The pod needs a Role allowing `get`, `create` and `update` on that one Lease (Task 10's manifests).

- [ ] **Step 1: Write the failing tests** against `tests/support/fake_api.rs`, a small in-test HTTPS server
  (rcgen cert) that implements `GET`, `POST` and `PUT` of one Lease, with `resourceVersion` conflicts (409) and a
  switch that makes renewals fail:
  - `acquires_missing_lease`.
  - `second_pod_waits_then_takes_over_after_lease_duration`; `leaseTransitions` increments.
  - `conflict_on_update_retries_without_claiming_leadership`.
  - `renewal_failures_drop_leadership_at_renew_deadline`.
  - `step_down_lets_another_pod_take_over_at_once`.
  - `cargo tree -p speedyspec-server-infra-kubernetes -i aws-lc-sys` prints nothing (a CI step).
- [ ] **Step 2: Run** `cargo test -p speedyspec-server-infra-kubernetes` → FAIL.
- [ ] **Step 3: Implement**, and wire `leadership = "kubernetes"` in the binary.
- [ ] **Step 4: Run** the whole suite → PASS.
- [ ] **Step 5: Commit.** `feat(infra-kubernetes): leadership through a coordination.k8s.io Lease`

---

### Task 10: Leader and followers across processes, and on Kubernetes

**Files:**
- Modify: `crates/speedyspec-server/tests/cluster.rs`
- Create:
  - `crates/speedyspec-server/tests/kubernetes_e2e.rs`;
  - `scripts/k3s.sh`, copied from `packages/speedyspec-cli/scripts/k3s.sh` with its own container name, label and
    port;
  - `Dockerfile` (static musl binary, `FROM scratch`);
  - `deploy/kubernetes/{serviceaccount,role,rolebinding,pvc,deployment,service}.yaml`.
- Modify: `.github/workflows/ci.yml` (a `kubernetes` job)

**Interfaces:**
- Consumes: Tasks 7–9.
- Produces:
  - `deploy/kubernetes/`:
    - a 3-replica Deployment with `POD_NAME` from the downward API, the probe on port 9090 wired to
      `readinessProbe` and `livenessProbe`, and `data_dir` on the PVC;
    - a Role with `get`, `create` and `update` on `leases/speedyspec-server`;
    - its RoleBinding and the ServiceAccount.
  - The README says the PVC must be ReadWriteMany for pods on several nodes. The test cluster has one node, so its
    `local-path` volume (ReadWriteOnce, shared by the pods on that node) stands in (ruling: k3s's single node has no
    RWX provisioner).
- [ ] **Step 1: Write the failing tests.**
  - `tests/cluster.rs`, the spec's "Leader and followers" list over three processes with `leadership = "file"`:
    - `two_step_switch_lands_on_all_three_within_one_poll`;
    - `member_failing_prepare_goes_not_ready_others_switch` (a `/readyz` 503 on that pod);
    - `killing_leader_mid_switch_hands_over` (SIGKILL the leader's keeper between staged and active → another pod
      leads within lease duration plus retry period, and finishes or abandons the switch);
    - `deposed_leader_late_write_ignored` (pause the leader process with SIGSTOP past its renew deadline, let another
      take over and write, then SIGCONT → its next write is refused, and nothing it wrote is served);
    - `follower_refuses_release_commands_with_leader_name`.
  - `tests/kubernetes_e2e.rs` (`#[ignore]`; kubeconfig only from `SPEEDYSPEC_E2E_KUBECONFIG`, never the user's own,
    as the CLI's e2e test does):
    - build and import the image;
    - apply `deploy/kubernetes/` with a test `ReleaseHost` reachable from the cluster;
    - `admin_switch_reaches_all_replicas` (each pod's `/` through `kubectl port-forward` serves the new version);
    - `server_upgrade_rolls_all_replicas` (every pod's `status` ends on 0.2.0 confirmed; the Lease's holder never
      changed during the rollout).
- [ ] **Step 2: Run** `cargo test -p speedyspec-server --features test-hooks --test cluster`. Tasks 3–8 built and
  unit-tested each piece, so most of these may pass on the first run; that is expected, and it is recorded in the
  ledger. Each failure is a bug: reproduce it with a test in the owning crate that fails first, then fix it there.
- [ ] **Step 3: Run** the e2e locally: `scripts/k3s.sh up`, then
  `SPEEDYSPEC_E2E_KUBECONFIG=… cargo test -p speedyspec-server --features test-hooks --test kubernetes_e2e --
  --ignored --test-threads=1`, then `scripts/k3s.sh down` → PASS.
- [ ] **Step 4: CI.** Add a `kubernetes` job on `ubuntu-latest` that runs `k3s.sh up`, the e2e test and
  `k3s.sh down` (`if: always()`). Run the whole suite on macOS and Linux musl → PASS.
- [ ] **Step 5: Commit.** `test: leader and followers across processes, and a three-replica Kubernetes end to end`

---

### Task 11: Docs for operators

**Files:**
- Create: `docs/cluster.md`
- Modify: `README.md`, `docs/handoff.md`
- Test: none new; `readme_example_config_parses` covers a `[cluster]` table added to the README's example (as a
  second `toml` block, which the test is extended to parse too).

- [ ] **Step 1: Write `docs/cluster.md`.** Cover:
  - the roles: leader, follower, member files;
  - the two-step switch, with the events it logs;
  - liveness by heartbeat counter;
  - epochs and how a deposed leader's late write is ignored;
  - the rollout order and revert;
  - what a new leader does;
  - the Lease and its Role;
  - why there is no `flock` on the shared volume.
- [ ] **Step 2: Update the README.**
  - Add a "Kubernetes (managed, several replicas)" section: the `[cluster]` keys, `deploy/kubernetes/`, probes, the
    ReadWriteMany requirement, and that release commands run on the leader (followers name it).
  - Add sample `status` output with `cluster` and `member` lines, taken from a real Task 10 run.
- [ ] **Step 3: Run** the whole suite on macOS and Linux musl → PASS.
- [ ] **Step 4: Commit.** `docs: running a cluster, and how leaders, followers and rollouts work`
