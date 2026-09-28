# Fusion Chaos/Stress Test Plan

Tracking doc for chaos and pressure testing on top of `test_fusion_simple.yml`. Scope for now: **KV/data-service nodes only** — no index/query/fts/analytics services in the fusion scope yet, so chaos targeting those daemons (the bulk of what `tests/2i/totoro/test_gsi_totoro.yml` does) is out of scope until a fusion scope adds those services.

## Approach

One test file per scenario, not one combined file — matches the repo's dominant pattern (`tests/2i/neo/`: 6 files, `tests/xdcr/`: 7 files, each scoped to one concern) rather than the `totoro` everything-in-one-file outlier. All share `scope_fusion_simple.yml` and the existing `tests/templates/{fusion,nfs}.yml` templates.

## Planned tests

- [ ] **`test_fusion_memcached_pause.yml`**
  Pause memcached (`memcached_kill`, SIGSTOP) → wait → resume (`start_memcached`, SIGCONT) → *then*, once confirmed healthy, trigger `fusion_rebalance`. Not concurrent with the rebalance — see "Ruled out" below for why.
  Precedent: `tests/rebalance/test_allRebalance_without_durability.yml:157-172`, `tests/xdcr/test_xdcrStress.yml:250-262` — both pause/resume memcached in isolation, then explicitly re-trigger rebalance as a separate step.

- [x] **`test_fusion_node_outage.yml`** — built.
  Tier-2 chaos, **no topology change**: autofailover left disabled (or its timeout set longer than the outage), `cbutil`/`cbinit.py` full `systemctl stop couchbase-server.service` on a KV node, wait, `systemctl start` the same node back. The node never leaves the cluster — no failover, no rebalance, no `fusion_rebalance` at all. Checks whether Fusion's uploader/sync state recovers cleanly from the *simplest* possible fault (the node's own service crashing and coming back), isolated from anything failover-related. This is deliberately the cheaper, more basic check to run before `test_fusion_autofailover_single_node.yml` below — if this one fails, the problem is in Fusion's basic crash-recovery path, not anything failover-specific, and the two failure causes shouldn't be conflated by only running the harder test.
  Precedent: `containers/cbutil/cbinit.py` for the stop/start mechanism; no existing Sequoia test does the "stop and restart in place, no failover" shape specifically, so this is adapted from the mechanism, not a direct precedent.

  **As built:** `scope_fusion_node_outage.yml` — 6 nodes total, 4 active (`init_nodes: 4`) + 2 spare for later `fusion_rebalance` use, KV mem quota `ram: 10500` (absolute MB, not `%`) per node, no `buckets:` block (created explicitly in the test file, after `enable_fusion`, same reasoning as `scope_fusion_simple.yml`). `test_fusion_node_outage.yml` — 10 Fusion-backed magma buckets (`bucket1`-`bucket10`, alternating 128/1024 vbuckets, 2 replicas). **Couchbase requires >= 1GB ram quota for any bucket with 1024 vbuckets** — the 5 buckets using 1024 vbuckets are sized 1024/1200/1500/1800/2048 MB accordingly, the 5 using 128 vbuckets go as low as 100 MB; total across all 10 is 10,072 MB, under the node's 10,500 MB quota. 1 scope × 10 collections per bucket (100 collections total, via `create-multi-scopes-collections`), loaded with `sequoiatools/magmaloader --all_coll true` per bucket. The outage itself targets `{{.ActiveDataNode 1}}` specifically — not the orchestrator, since every REST check in the test targets `{{.Orchestrator}}`, which must stay reachable throughout. Recovery is confirmed via `martin/wait` polling the restarted node's REST port (the same tool Sequoia's own `WaitForServers()` uses), then a **`fusion_rebalance` settle pass** — `$1`/`$2` (current/new node lists) are identical, so membership still doesn't change, but it exercises the full accelerated-rebalance pipeline (prepareRebalance → manifest split/sync → accelerator → start rebalance) as a real post-recovery check rather than just a REST status poll — followed by `get_fusion_status` + `get_active_guest_volumes` + a raw `/pools/default` check.

- [ ] **`test_fusion_abandoned_rebalance.yml`**
  `rebalance_stop` immediately after `fusion_rebalance` starts (interrupted, not completed, accelerated rebalance). Confirm cluster and Fusion guest-volume state recover cleanly, and that Fusion doesn't get stuck in a bad state from an abandoned accelerator run.
  Precedent: `tests/2i/totoro/test_gsi_totoro.yml:1152` uses `rebalance_stop` the same way for GSI.

- [ ] **`test_fusion_autofailover_single_node.yml`**
  Unlike `test_fusion_node_outage.yml` above, this one **does** change cluster membership, twice: `enable_autofailover` with a short timeout first, then the same `cbutil`/`cbinit.py` service stop on one KV/Fusion node — but this time the outage is left running long enough that Couchbase's own auto-failover actually evicts the node and `rebalance` removes it from the cluster, before it's restarted and explicitly re-added + rebalanced back in. Checks `/fusion/status` and uploader state through a genuine eviction/rejoin cycle, not just a service restart.
  Precedent: `tests/templates/multinode_failure.yml:200` (`autofailover1Node`) — same enable-autofailover → stop → sleep → rebalance → restart → re-add shape, adapted to check Fusion status instead of just node membership.

- [ ] **`test_fusion_autofailover_multi_node.yml`**
  Bring down 2 KV/Fusion nodes simultaneously, let auto-failover evict both, rebalance, restart, and add both back. Exercises Fusion recovery when more than one node's guest volumes go dark at once, not just single-node loss.
  Precedent: `tests/templates/multinode_failure.yml:138` (`autofailover2Nodes`) and `:4` (`multinodefailover`).
  `scope_fusion_simple.yml` currently provisions 2 nodes (1 active + 1 spare for the `fusion_rebalance` demo), but fusion scopes are meant to flex from 2-10 nodes — this test just needs to target a scope variant sized to keep quorum during a 2-node outage (5+ nodes, mirroring `autofailover3Nodes`'s shape).

- [ ] **`test_fusion_quorum_loss.yml`**
  Bring down a *majority* of nodes at once (e.g. 3 of 5) and confirm Couchbase's auto-failover safety check refuses to fail them all over, rather than doing so and leaving the cluster in an inconsistent state. This is the closest Couchbase equivalent to a "split-brain" test: Couchbase doesn't have Raft/multi-master-style split-brain since there's a single cluster orchestrator, but auto-failover has a built-in quorum guard (won't auto-failover a majority of nodes) that this test would exercise directly against Fusion-backed buckets.
  Precedent: none in-repo yet. `enable_autofailover`'s `--max-failovers` cap (`tests/templates/rebalance.yml:360`) is the underlying mechanism, but no existing Sequoia test drives it past quorum — this would be new ground, not an adaptation.
  Same scope note as above — needs a node count where failing a majority still leaves a defined minority, e.g. 5 nodes.

## Ruled out (with reasoning)

- **Kill/pause memcached *while* `fusion_rebalance` is actively running** (totoro-style mid-rebalance kill, adapted to KV). Checked real usage of `memcached_kill` in this repo — it's never used concurrently with an in-flight rebalance, always pause-then-resume-then-manually-retrigger. Unlike indexer/projector (which have their own rebuild/retry semantics under the rebalance orchestrator, which is why totoro can kill them mid-transfer), memcached is the core data-serving engine for the vbuckets actually being moved — if it goes down mid-DCP-stream, the rebalance has no way to resume seamlessly. No auto-retry in Couchbase itself, and `run_fusion_rebalance.py` has zero retry logic of its own (one `prepareRebalance` call, one `start_rebalance` call, no loop) — so this would just fail the rebalance outright with no interesting signal.

## Open questions

- **Does Fusion have its own separately-killable daemon/process** (the log-store uploader/sync mechanism), or does it run as threads inside memcached/ep-engine itself? If the latter, there may be no KV-safe way to do a totoro-style "kill the fusion-specific process mid-transfer" test at all on KV-only nodes. Needs checking against a live node (`ps aux` for anything fusion-named) — not answerable from the codebase alone.
- **Which scope size to target for multi-node chaos.** `scope_fusion_simple.yml` has 2 nodes today, but fusion scopes are planned to run anywhere from 2-10 nodes. `test_fusion_autofailover_multi_node.yml` and `test_fusion_quorum_loss.yml` just need to run against a scope sized so quorum survives the outage (likely 5+, mirroring `autofailover3Nodes`'s shape) — not a new capability, just picking/writing the right scope file for these two tests.

## In the plan: infrastructure-level chaos (CPU, RAM, disk, network)

Not started — flagging for visibility before scoping in detail. Checked `tests/templates/` and `containers/` repo-wide: **there is no CPU/RAM/disk-fill/network chaos tooling anywhere in Sequoia today** (same gap noted in [`tests/2i/totoro/CHAOS_AND_PRESSURE.md`](../2i/totoro/CHAOS_AND_PRESSURE.md), §02 — everything that exists there is process/service/membership-level too). This would be new capability for the framework, not just new Fusion test files.

Rough shapes to explore, none committed yet:
- **CPU / RAM pressure** — run `stress-ng` (or similar) against a node over SSH, the same `sequoiatools/cmd` + `sshpass` pattern `kill_process` already uses (`tests/templates/rebalance.yml:395`). Need to check whether `stress-ng` is available on the couchbase-server images (`containers/couchbase/*/Dockerfile`) or would need adding.
- **Disk pressure** — fill the Fusion NFS-backed volume (`fallocate`/`dd`) to exercise near-capacity/at-capacity behavior; throttle disk I/O if the storage driver allows it.
- **Network chaos** — latency/loss/partition between a KV node and the NFS server, or between cluster nodes, via `tc`/`netem` or a tool like `pumba`/`toxiproxy` against the Docker network. No existing precedent in this repo. The experimental `--network cbl` flag (`CLAUDE.md`) may be relevant plumbing to check first.

Open question for this section: whether chaos runs directly against the container network (needs root/`NET_ADMIN` inside the container) or needs a dedicated chaos container/sidecar — not resolved yet.

## Later stage: a `test_8.5.yml`-style comprehensive test

**Status: v1 built (`scope_fusion_comprehensive.yml` / `test_fusion_comprehensive.yml`), not yet run.** Originally framed below as "not a near-term item" pending two things — the 5 KV-only scenario tests existing first, and an answer to whether Fusion has anything distinctive to test at the other service layers. Explicitly chose to start basic now rather than wait on either, on the understanding this is a genuine first attempt (especially the eventing/analytics wiring) that will need a debugging pass once actually run, same as every other test in this suite has needed.

**What v1 actually is** — deliberately small, not the "8-12 buckets, 10+ nodes" version originally sketched below: 9 nodes (`count: 9, init_nodes: 8` — 4 plain KV, 1 each of analytics/eventing/index/query, 1 spare outside `init_nodes` for the `fusion_rebalance`), 3 buckets (`bucket1`/`bucket2` Fusion-backed magma, `metabucket` plain couchstore serving as eventing's metadata bucket + analytics' catapult-loaded data source), one GSI index + a couple of N1QL queries, one simple `sbm`-type eventing handler, one minimal analytics dataverse/dataset/index, then the same `fusion_rebalance`-in-a-spare-node pattern as `test_fusion_node_outage.yml` — all of it running concurrently, then torn down.

**Not reused directly**: `tests/eventing/CC/test_eventing_rebalance_integration.yml`'s `create_and_deploy` and `tests/analytics/cheshirecat/test_analytics_integration_scale3.yml`'s `analytics_setup` (the same sections `test_8.5.yml` composes via `test:`/`section:`) assume a `test_8.5`-shaped bucket layout (4+ specific bucket indices, `event_0.coll0-3` collection wiring) that doesn't fit this smaller scope — v1 adapts their flag shapes onto its own 3 buckets by hand instead. Revisit reusing the real sections once this either proves out or the bucket layout grows to match.

**Still true and worth remembering**: the original open question — whether Fusion interacts with GSI/FTS/N1QL/Eventing/Analytics/XDCR in some Fusion-specific way, or those services just need to not break on top of a Fusion-backed cluster — is answered empirically by what breaks (or doesn't) once this actually runs, not resolved in advance. Treat the first real run's results as that answer, not as a bug list to silently fix.

## Out of scope until index/query services are added

- Indexer/projector/cbq-engine kill mid-rebalance (totoro's core technique) — no such services on KV-only nodes yet.
- `autofailover2Nodes`/`3Nodes` (multi-node simultaneous outage) — start with single-node once the above 3 are solid.
