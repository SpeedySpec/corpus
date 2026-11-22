# Cluster

`speedyspec-server-app` (`releases.rs`, `cluster.rs`), `speedyspec-server-infra-fs`,
`speedyspec-server-infra-kubernetes`, `speedyspec-server`, `deploy/kubernetes`. From the cluster plan (#3).

## A leader deposed mid-download still commits the release into releases/

- **Where:** `speedyspec-server-app`, `releases.rs`, the install path (`store.commit`, then `commit_state`).
- **What happens:** a leader that loses the Lease while downloading still unpacks and renames the release into
  `releases/`. Only the state save afterwards refuses. The directory is left in the shared store with no state entry
  pointing at it.
- **Suggested fix:** check leadership again before `store.commit`, and discard the partial if it is lost.

## Member files of departed pods are never deleted

- **Where:** `speedyspec-server-infra-fs`, `members.rs`.
- **What happens:** each pod writes a member file named after itself. Deployment pod names are random, so every
  replaced pod leaves a file behind, and nothing deletes them. `status` lists them all, and a newly started leader
  process counts them as live for the first 10 s (`STALE_AFTER`).
- **Suggested fix:** have the leader delete member files that have been stale for much longer than `STALE_AFTER`.

## A leader whose own prepare failed stays not ready

- **Where:** `speedyspec-server` `cluster.rs` (the leader runs `lead`, never `follow`); `releases.rs`,
  `finish_staged`.
- **What happens:** when the leader fails to prepare a staged set but other pods succeed, the set still goes active.
  The leader never runs `follow`, so it never tries to prepare the active set again, and it stays not ready until
  leadership moves to another pod.
- **Suggested fix:** let the leader retry preparing the active set it does not serve, as a follower does.

## finish_staged and take_over hold the state lock while they wait

- **Where:** `speedyspec-server-app`, `releases.rs`, `finish_staged` (called with the state guard) and `take_over`.
- **What happens:** the leader waits up to `prepare_timeout` for other pods to report, while holding the state lock.
  `pin`, `rollback`, repair and the next leader step all block until it ends.
- **Suggested fix:** release the lock while waiting, and check the revision again when taking it back.

## A pod that boots slowly goes stale in the middle of its rollout turn

- **Where:** `speedyspec-server-app`, `cluster.rs` (`STALE_AFTER` = 10 s) and the rollout logic in `releases.rs`.
- **What happens:** during a server rollout, a pod whose new generation takes more than 10 s to start stops reporting
  long enough to be counted as gone. The leader can then start the next pod's turn while the first is still on
  trial.
- **Suggested fix:** keep a pod in its turn until it reports again or a turn timeout passes, rather than relying on
  liveness alone.

## Two generations of one pod renew the same Lease

- **Where:** `speedyspec-server-infra-kubernetes`, `lib.rs` (the holder is the pod name).
- **What happens:** during a handoff, the old and new generation of one pod both run a leadership loop under the same
  holder name. They can conflict on updates, and both can believe they lead. An operation that started in the old
  generation before its drain can still commit at the same epoch.
- **Suggested fix:** include the generation in the holder identity, or stop the old generation's leadership loop
  (and in-flight leader work) when it starts draining.

## role.yaml fixes resourceNames to the default lease name

- **Where:** `deploy/kubernetes/role.yaml`, `docs/cluster.md`.
- **What happens:** the Role allows `get` and `update` only on a Lease named `speedyspec-server`. If
  `cluster.lease_name` is changed, leadership fails with permission errors. The docs do not say the Role must change
  with it.
- **Suggested fix:** say so in `docs/cluster.md` and next to `resourceNames` in `role.yaml`.

## step_down is not called on a real SIGTERM

- **Where:** `speedyspec-server`, `cluster.rs` and the generation's shutdown; `step_down` exists in both leadership
  adapters.
- **What happens:** when a leader pod is stopped, it does not release the Lease. The other pods must wait for it to
  expire, so each replacement of the leader pod leaves about 15 s without a leader.
- **Suggested fix:** call `step_down` when the generation shuts down for good (not on a handoff).

## FileLeaseLeadership and pause_renewals are compiled into production builds

- **Where:** `speedyspec-server-infra-fs`, `lease.rs`; `speedyspec-server`, `cluster.rs` and `config.rs`.
- **What happens:** the file-based lease and its `pause_renewals` test switch are built into every binary. Only the
  config refuses `leadership = "file"` without the `test-hooks` feature.
- **Suggested fix:** put them behind a feature, so release builds do not contain them.

## follow() retries a failing prepare on every poll

- **Where:** `speedyspec-server-app`, `releases.rs`, `follow`.
- **What happens:** a follower that cannot prepare the active set tries again on every poll interval and logs
  `active_set_not_prepared` each time. The log fills with the same error, and the prepare work repeats.
- **Suggested fix:** back off between attempts and log a repeated failure once.
