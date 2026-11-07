# Proposed Architecture

## Status

This document describes a proposed architecture. No components in this document have been implemented.

## Components

### Cache Nodes

Each cache node will own a subset of keys, store entries in memory, and expose operations for reading, writing, deleting, and exchanging replica data. A node should know the current placement view needed to determine whether it is a primary or replica for a key. Cache entries are expected to remain non-durable in the first version; an eviction, restart, or total replica loss can therefore produce a cache miss.

### Coordinator

The coordinator is proposed as the source of cluster membership and topology information. It will track registered nodes and their health state, publish a topology version, and initiate placement changes when membership changes. The precise availability model of the coordinator, whether it is a single service or replicated, is intentionally unresolved; it must not silently become an unexamined single point of failure.

### Consistent Hashing and Virtual Nodes

The coordinator and nodes will calculate a deterministic hash ring. Each physical cache node receives multiple virtual nodes on that ring. A key hashes to a point on the ring; its primary owner is the next virtual node clockwise. Virtual nodes are proposed to improve distribution when physical node counts are small and to limit movement after membership changes.

The hash algorithm, virtual-node count, collision handling, and ring-version distribution protocol are open decisions. Every participant must use the same versioned placement view to prevent disagreement about ownership.

### Replication

For a key, the next distinct physical nodes clockwise after the primary are candidate replica owners. A write may be sent to the primary and then replicated, or a client may write to several owners directly. The replication factor, acknowledgement policy, read policy, conflict rules, and expiration propagation are not decided yet.

### Health Monitoring and Failure Detection

The coordinator is proposed to collect periodic heartbeats or active health checks from nodes. A node that misses an as-yet-undecided threshold would move through a suspected state before being declared failed, preventing a single delayed message from triggering rebalancing. Network partitions and coordinator outages require an explicit policy because they can cause different participants to observe different membership views.

### Node Recovery and Data Rebalancing

When membership changes, the coordinator will compute a new ring version and identify ranges whose primary or replica owners change. Surviving owners can transfer eligible entries to new owners while continuing to serve requests according to a transition policy. A recovered node must not serve stale data immediately merely because it is reachable; it needs an explicitly designed synchronization and catch-up procedure before it is considered healthy for ownership.

### Metrics and Monitoring Dashboard

Nodes and the coordinator are proposed to expose operational metrics: request count and latency, hit/miss rate, memory use, entry count, replication outcomes, health transitions, topology version, and rebalance progress or failures. A dashboard can visualize these metrics and cluster membership. The metrics backend, dashboard technology, retention, and alert thresholds remain unselected.

## Request and Data Flow

1. A client obtains the current topology from the coordinator, directly or through a client library, and calculates the key's primary and replica owners from the versioned hash ring.
2. For a read, the client or routing layer sends the request to the primary owner. If the chosen read policy permits it, a replica can be tried after a failure or for load distribution.
3. For a write, the client sends the entry to the primary owner. The selected replication protocol then propagates it to replica owners and determines when the caller receives success.
4. Nodes report liveness and selected metrics to the coordinator. The coordinator publishes a new topology version when it accepts a membership change.
5. During a join, failure, or recovery, current and newly assigned owners transfer affected key ranges according to the rebalance protocol. Requests must be routed using a transition-safe topology view until transfer completes.
6. Metrics flow from nodes and the coordinator to the monitoring system; the dashboard reads that operational data rather than participating in request routing.

## Boundary and Invariants

The cache is not the authoritative store. Cache misses, expiry, or complete replica loss must be recoverable by fetching from an application-defined source of truth. The implementation should preserve these proposed invariants: topology versions are monotonic, placement is deterministic for a given version, writes follow an explicit acknowledgement rule, and a node is not assigned serving responsibility before its membership state permits it.
