# Proposed Architecture

## Status

This document describes a proposed architecture. No components in this document have been implemented.

## Components

### Cache Nodes

Each cache node will own a subset of keys, store entries in memory, and expose operations for reading, writing, deleting, and exchanging replica data. A node knows the current placement view needed to determine whether it is primary or replica for a key. Its proposed states are `HEALTHY`, `SUSPECT`, `FAILED`, and `RECOVERING`. Cache entries remain non-durable: eviction, restart, or loss of both copies can produce a cache miss, and the backing database remains the source of truth.

### Coordinator

Version 1 proposes one coordinator as the source of cluster membership and topology information. It tracks registered nodes and their health state, publishes monotonically increasing topology versions and fencing epochs, issues time-bounded topology leases, and serializes membership changes. This is intentionally a single point of failure for cluster management: while it is down, it cannot decide failures, issue failovers, or start rebalancing. Coordinator restart and recovery of authoritative topology state remain open questions.

### Consistent Hashing and Virtual Nodes

The coordinator and nodes will calculate a deterministic hash ring. Each physical cache node initially receives 64 virtual nodes on that ring. A key hashes to a point on the ring; its primary owner is the next virtual node clockwise. Virtual nodes improve distribution when physical node counts are small and limit movement after membership changes. The value 64 is an experimental initial setting, not a demonstrated optimum.

The hash algorithm, collision handling, and ring serialization protocol remain open. Every participant must use the same versioned placement view to prevent disagreement about ownership. The next distinct physical node clockwise is the proposed replica owner, producing a replication factor of two.

### Replication

For a key, the next distinct physical node clockwise after the primary is its sole Version 1 replica. Writes are primary-led and synchronous: `SET` succeeds only when both primary and replica acknowledge storage of the same record. Reads route to the current primary. This protects acknowledged writes from a single cache-node failure if the replica acknowledgement is genuine, but it raises write latency and makes a write unavailable when the replica cannot acknowledge. It is neither permanent durability nor protection from loss of both copies.

Each record will carry a version and absolute `expiresAt` timestamp; deletes create versioned tombstones. Absolute expiration gives all copies the same intended expiry time but requires reasonably synchronized clocks. Tombstones prevent stale replicas from resurrecting deleted values. Version generation after primary promotion and safe tombstone cleanup remain open questions.

### Health Monitoring and Failure Detection

The coordinator collects one heartbeat per second from nodes. Three missed heartbeats move a node to `SUSPECT`; five move it to `FAILED`, at which point a serialized topology transition may begin. The suspicion interval reduces false positives at the cost of slower recovery.

Write authority requires two conditions: a node has the current topology/fencing epoch, and it holds an unexpired lease issued for that epoch by the coordinator. A newer epoch fences an old owner, while the lease limits how long an isolated old owner can continue writing if it has not yet received the newer topology. A version number alone cannot prevent split brain because a partitioned node may keep using its previously valid version. Under a partition, a node that cannot renew its lease must stop accepting writes; this deliberately reduces availability to protect ownership safety.

### Node Recovery and Data Rebalancing

When membership changes, the coordinator computes a new ring version and identifies ranges whose primary or replica owners change. Version 1 handles one topology change at a time. For an affected range, it pauses writes, transfers non-expired records and tombstones from current owners, verifies receipt, then publishes the transition that enables the new owner. Pausing writes makes the migration easier to reason about but reduces availability for affected keys.

A restarted node enters `RECOVERING`. It cannot serve traffic or receive ownership until required data synchronization completes and the coordinator marks it eligible. If transfer is interrupted, the node remains `RECOVERING`; it must not be activated with a partial range. The exact resumable/idempotent transfer protocol remains open.

### Metrics and Monitoring Dashboard

Nodes and the coordinator are proposed to expose operational metrics: request count and latency, hit/miss rate, memory use, entry count, replication outcomes, health transitions, topology version, and rebalance progress or failures. A dashboard can visualize these metrics and cluster membership. The metrics backend, dashboard technology, retention, and alert thresholds remain unselected.

## Request and Data Flow

1. A client obtains the current topology from the coordinator, directly or through a client library, and calculates the key's primary and replica owners from the versioned hash ring.
2. For a read, the client or routing layer sends the request to the current primary owner.
3. For a write, the client sends the entry to the primary owner. The primary verifies its current lease and fencing epoch, stores the record, waits for replica acknowledgement, and only then reports success.
4. Nodes report liveness and selected metrics to the coordinator. The coordinator publishes a new topology version when it accepts a membership change.
5. During a join, failure, or recovery, current and newly assigned owners transfer affected key ranges according to the rebalance protocol. Requests must be routed using a transition-safe topology view until transfer completes.
6. Metrics flow from nodes and the coordinator to the monitoring system; the dashboard reads that operational data rather than participating in request routing.

## Boundary and Invariants

The cache is not the authoritative store. Cache misses, expiry, or complete replica loss must be recoverable by fetching from an application-defined source of truth. The implementation should preserve these proposed invariants: topology versions and fencing epochs are monotonic, placement is deterministic for a given version, writes follow the two-acknowledgement rule and require a valid lease, and a node is not assigned serving responsibility before its membership state permits it.

## How We Will Test the Proposed Architecture

- Kill a primary immediately after a successful `SET` and verify the acknowledged record is available after promotion.
- Make a replica unavailable and verify the primary does not falsely acknowledge a synchronous write.
- Verify identical expiry behavior on both copies and that tombstones stop stale-value resurrection.
- Drop heartbeats to assert the three-miss `SUSPECT` and five-miss `FAILED` transitions.
- Partition an old owner from the coordinator, let its lease expire, and verify it rejects writes after a newer owner is fenced in.
- Interrupt recovery transfer and verify the node remains `RECOVERING` and cannot serve traffic.
- Move a range and verify writes to that range pause, transferred records remain readable after activation, and unrelated ranges remain writable.
- Add or remove a node and verify deterministic placement, distinct replicas, one serialized transition, and bounded movement for the initial 64-virtual-node ring.

## Open Questions and Future Improvements

- How should promoted primaries generate versions that order correctly after failover?
- When can tombstones be safely garbage-collected?
- How should interrupted transfers resume or restart idempotently?
- How does a restarted coordinator reconstruct authoritative membership, epochs, leases, and topology?
- What lease duration, heartbeat timeouts, and virtual-node count are suitable after measurement?
- Should a later version add coordinator replication, snapshot-plus-delta migration, or replica-read consistency options?
