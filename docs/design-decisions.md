# Proposed Design Decisions

## Status

All entries describe proposed Version 1 choices and remaining investigations, not implemented behavior.

## Proposed Version 1 Summary

Version 1 proposes one primary and one replica on distinct physical nodes, primary-led synchronous replication, primary reads, one coordinator, and consistent hashing with 64 virtual nodes per physical node. Topology changes are serialized. A restarting node remains `RECOVERING` until synchronization completes; migrations pause writes only for affected ranges. These constraints favor teachability and explicit correctness boundaries over maximum availability or throughput.

## Runtime and Service Interface

**Problem:** The project needs an accessible implementation environment and a way for clients and cluster members to communicate.

**Proposed approach:** Use TypeScript on Node.js and begin with HTTP service interfaces.

**Why consider it:** TypeScript offers a low-friction environment for a learning-oriented distributed-systems project, while HTTP is inspectable with common tools and straightforward to test.

**Alternatives:** Go with gRPC or HTTP; Java with a mature distributed-systems ecosystem; Rust for tighter resource control; Node.js with a binary RPC protocol.

**Trade-offs:** TypeScript and HTTP favor iteration and transparency but add runtime overhead and may be less suitable for high-throughput production workloads. A binary protocol improves efficiency at the cost of observability and implementation complexity.

**Questions to investigate:** Which framework, validation library, timeouts, and error format fit the API? Should external clients and node-to-node traffic use different protocols?

## Key Placement

**Problem:** Keys must be distributed predictably while minimizing movement when cluster membership changes.

**Proposed approach:** Use consistent hashing with 64 virtual nodes per physical cache node initially, with the next distinct physical node clockwise chosen as replica.

**Why consider it:** It supports deterministic routing and usually moves only affected hash ranges as nodes join or leave. Virtual nodes can smooth uneven distribution.

**Alternatives:** Modulo hashing; rendezvous hashing; fixed partition tables; a coordinator-selected routing map.

**Trade-offs:** A hash ring is conceptually manageable but introduces ring-version coordination and virtual-node tuning. The initial value of 64 is experimental rather than proven. Rendezvous hashing can simplify membership calculations but has different computational and replication characteristics.

**Questions to investigate:** Which stable hash function should be used? Does 64 virtual nodes provide acceptable balance and movement? How is the ring serialized and versioned?

## Coordinator Role and Availability

**Problem:** Participants need an authoritative membership and topology view.

**Proposed approach:** Use one coordinator for Version 1. It manages membership, heartbeats, recovery, serialized topology changes, leases, and monotonically increasing fencing epochs/topology versions.

**Why consider it:** Centralizing the initial control plane makes membership transitions and their observability easier to understand.

**Alternatives:** Gossip-based membership; an external coordination system; a replicated consensus-backed coordinator; static membership for the earliest prototype.

**Trade-offs:** A single coordinator simplifies the initial build but is a single point of failure for cluster management. Coordinator loss stops authoritative membership changes and rebalancing. Distributed membership improves resilience but adds disagreement, convergence, and split-brain concerns.

**Questions to investigate:** How does a restarted coordinator recover authoritative topology state, epochs, and leases? What can nodes safely do during coordinator unavailability? Does the coordinator need replication before failure automation is introduced?

## Replication Consistency

**Problem:** Replicas improve availability, but reads and writes can observe different versions during delay, failure, or rebalancing.

**Proposed approach:** Use replication factor 2, with a primary and replica on distinct physical nodes. The primary synchronously replicates each `SET` and reports success only after both nodes acknowledge storage. Reads route to the current primary.

**Why consider it:** Primary-led replication gives a clear ownership path and makes the first replication flow easier to reason about.

**Alternatives:** Synchronous replication to all replicas; quorum reads and writes; asynchronous replication; client-side fan-out; no replication in the first executable milestone.

**Trade-offs:** Two acknowledgements protect an acknowledged write from one cache-node failure if the replica acknowledgement is genuine. They do not provide permanent durability or protect against loss of both copies. Synchronous replication increases latency and rejects writes when the replica cannot acknowledge. Asynchronous replication is faster but can lose recently acknowledged cache writes. Quorums require more nodes, versioning, and conflict handling.

**Questions to investigate:** How are concurrent writes ordered or resolved? How should version generation work after primary promotion? How should client operation IDs make retries idempotent when a timeout follows a successful write? What replica-read behavior, if any, belongs in a later version?

## Record Correctness: Versions, TTLs, and Tombstones

**Problem:** Replicas can receive delayed data, expire keys differently, or retain an old value after a delete.

**Proposed approach:** Store a record version and absolute expiration timestamp with each value. Represent `DELETE` as a versioned tombstone and replicate it synchronously with ordinary writes.

**Why consider it:** Versions let nodes reject older updates. A shared absolute expiry gives replicas the same intended deadline. Tombstones prevent stale data from reappearing after a node recovers or receives delayed traffic.

**Alternatives:** Propagate remaining TTL only; physically remove deletes immediately; use eventual anti-entropy to repair stale replicas.

**Trade-offs:** Absolute timestamps require reasonably synchronized clocks. Tombstones consume memory and need eventual cleanup. Version generation after primary promotion and safe tombstone cleanup are intentionally unresolved.

**Questions to investigate:** What clock-skew bound is acceptable locally? How are post-promotion versions generated? When can every relevant replica be known to have observed a tombstone?

## Failure Detection Thresholds

**Problem:** The cluster must distinguish a failed node from a slow node or a temporary network partition.

**Proposed approach:** Send a heartbeat every second. The coordinator marks a node `SUSPECT` after three missed heartbeats and `FAILED` after five. The other states are `HEALTHY` and `RECOVERING`.

**Why consider it:** It is observable and understandable for an initial coordinator-based model, and a suspicion period can reduce unnecessary rebalancing.

**Alternatives:** Active probes only; phi-accrual detection; gossip-based failure detection; immediate removal after a missed heartbeat.

**Trade-offs:** Aggressive thresholds react faster but risk false positives and churn. Conservative thresholds preserve stability but increase time spent routing around a genuinely failed node. No detector can reliably distinguish all failures from partitions.

**Questions to investigate:** Are the initial one-, three-, and five-second parameters acceptable under local scheduling delays? How are partitions represented? What metrics validate false-positive and detection-delay behavior?

## Write Fencing and Leases

**Problem:** A partitioned former primary can continue accepting writes after the coordinator assigns its range to another node.

**Proposed approach:** Require a valid topology lease and current fencing epoch for every write. The coordinator increments epochs with topology changes and issues leases only to owners under that epoch.

**Why consider it:** A newer epoch causes recipients to reject operations from an old owner. The lease bounds the time an isolated node can continue writing when it has not yet learned about that newer epoch. Both are necessary: a topology version alone does not revoke authority from a node that cannot receive the new version.

**Alternatives:** Accept split-brain writes and reconcile later; permanently route all writes through the coordinator; use a replicated consensus system for ownership.

**Trade-offs:** Lease renewal ties write availability to coordinator reachability and clock assumptions. This deliberately favors ownership safety during partitions over always accepting writes.

**Questions to investigate:** What lease duration and renewal margin are safe? How are clock skew and coordinator restart handled? How are epochs persisted or recovered?

## Recovered-Node Synchronization

**Problem:** A previously failed node can return with missing or stale memory state and must rejoin without corrupting placement or serving stale values.

**Proposed approach:** A restarted node enters `RECOVERING`, registers, receives the current topology, synchronizes assigned ranges from active owners, verifies completion, and only then becomes eligible to serve traffic.

**Why consider it:** Separating reachability from serving eligibility prevents a restarted node from immediately advertising incomplete or obsolete data.

**Alternatives:** Discard all local state and warm from misses; immediately rejoin and repair lazily on reads; use anti-entropy continuously; persist cache data locally between restarts.

**Trade-offs:** Eager synchronization improves readiness and predictability but consumes network and memory capacity. If transfer is interrupted, the node remains `RECOVERING` and unavailable until a safe retry or restart occurs. Lazy repair reduces recovery time but permits more misses or stale reads.

**Questions to investigate:** Is local recovered state discarded, reconciled, or version-checked? How is synchronization made idempotent and resumable after interruption? What verification makes the node safe to receive primary traffic?

## Serialized Migration

**Problem:** Moving ownership while writes continue can omit updates or produce conflicting owners.

**Proposed approach:** Serialize topology changes. For each affected range, pause writes, transfer records and tombstones, verify the destination, and activate the new placement.

**Why consider it:** Pausing affected writes avoids the first-version complexity of a change log or snapshot-plus-delta protocol.

**Alternatives:** Continue writes and replay a mutation log; dual-write to old and new owners; snapshot with a delta catch-up phase.

**Trade-offs:** The approach is easier to verify but temporarily reduces write availability for affected keys and slows unrelated membership changes by processing one topology transition at a time.

**Questions to investigate:** How are pauses signaled to clients? What timeout aborts a stalled migration? How should a later version perform concurrent, resumable migration?

## Observability

**Problem:** Self-healing behavior cannot be trusted or debugged without visibility into topology, failures, and data movement.

**Proposed approach:** Expose structured logs and Prometheus-compatible metrics from all control-plane and data-plane services, with a dashboard added after metric semantics stabilize.

**Why consider it:** Metrics make recovery behavior measurable and support automated tests that assert state transitions.

**Alternatives:** Logs only; OpenTelemetry-first tracing; a hosted observability service; custom event storage.

**Trade-offs:** Metrics are efficient for aggregate behavior but may omit per-request history. Tracing improves causal diagnosis but adds instrumentation and collection complexity.

**Questions to investigate:** Which labels avoid excessive cardinality? Which signals define rebalance health? What retention and alerting are appropriate for a local development cluster?

## How We Will Test the Proposed Decisions

- Replication: crash a primary after successful acknowledgement and verify its replica retains the value; make the replica unavailable and verify no false write success.
- Record correctness: delay old messages, test matched absolute expiry, and prove a tombstone prevents stale-value resurrection.
- Placement: assert deterministic primary/replica selection, distinct physical owners, bounded movement, and distribution for the experimental 64-virtual-node configuration.
- Failure detection: suppress heartbeats and assert the exact `HEALTHY` to `SUSPECT` to `FAILED` progression.
- Fencing: isolate an old owner, expire its lease, advance the epoch, and verify it rejects writes while the current owner accepts them.
- Recovery: interrupt a transfer and verify the node remains `RECOVERING`; complete it and verify only then can it serve.
- Migration: assert that only the affected range pauses writes, all transferred records survive cutover, and a second topology transition waits for the first.
- Coordinator: stop and restart it, then characterize safe read/write behavior and validate the eventual chosen topology-recovery procedure.

## Open Questions and Future Improvements

- Operation IDs and idempotency for client retries after ambiguous timeouts.
- Version generation after primary promotion.
- Safe tombstone cleanup.
- Clock synchronization bounds for absolute TTLs and leases.
- Interrupted-transfer retry, resume, and validation semantics.
- Coordinator restart and authoritative topology-state recovery.
- Measured tuning of heartbeat thresholds, lease duration, and virtual-node count.
- Later support for replicated coordination, concurrent migration, quorum semantics, and replica reads.
