# Self-Healing Distributed Cache System

## Project Status

Stage 1 provides a Spring Boot application foundation and an application-context test. No cache, coordinator, dashboard, or infrastructure features have been implemented.

## Stage 1 Foundation

The executable foundation uses Java 17, Spring Boot 4.1.2, Maven, Spring Boot WebMVC, and JUnit 5 through Spring Boot Test. The planned cache behavior remains unimplemented at this stage.

Run the test suite on Windows with:

```powershell
.\mvnw.cmd test
```

Start the application locally with:

```powershell
.\mvnw.cmd spring-boot:run
```

The application starts on port `8080`. No HTTP endpoints have been added yet.

## Overview

A distributed cache stores frequently accessed data in memory across multiple networked machines rather than on one server. Applications use it to reduce latency and lower load on an authoritative data store, while the cache cluster provides more capacity and availability than a single process can provide.

This project will explore a distributed cache that continues to serve and restore data placement when cache nodes join, leave, or fail. It targets the operational gaps of a simple multi-node cache: routing requests predictably, retaining useful data after a node loss, detecting unhealthy members, and rebalancing data without an operator manually repairing the cluster.

In this context, self-healing means the system is designed to detect a node failure, update membership and routing, preserve or restore replicas where possible, and incorporate recovered or new nodes safely. It does not imply that every failure can be healed or that data is durable without an authoritative backing store.

## Planned Features

- Key-based `GET`, `SET`, and `DELETE` cache operations.
- Coordinator-assisted node membership and client routing.
- Consistent hashing with virtual nodes for balanced key placement.
- Configurable replication of cache entries.
- Health checks, failure detection, and membership changes.
- Automatic rebalancing after node joins, failures, and recovery.
- Metrics, structured logs, and a monitoring dashboard.
- Failure-focused integration and recovery testing.

These are proposed features, not implemented functionality.

## Proposed Technology Stack

- Runtime and language: Node.js with TypeScript.
- Service interfaces: HTTP with a lightweight TypeScript web framework, selected during implementation.
- Cache-node storage: in-memory data structures for the first version.
- Coordination and membership: a dedicated coordinator service using an initially simple, explicitly documented membership model.
- Observability: Prometheus-compatible metrics, structured logs, and a browser dashboard considered for a later phase.
- Testing: a TypeScript test runner, HTTP integration tests, and multi-process fault-injection scenarios.

The stack is provisional. It is intended to make the first implementation approachable while keeping cluster behavior observable; dependencies and exact frameworks have not been selected or installed.

## Proposed Version 1 Baseline

Version 1 is a proposed design, not implemented functionality. It uses a replication factor of two: one primary and one replica on distinct physical nodes. A `SET` succeeds only after both nodes acknowledge the same versioned record. This protects an acknowledged write from one cache-node failure when the replica acknowledgement is genuine. It does not make the cache durable, and it cannot protect data when both copies are lost. The application's backing database remains the source of truth.

Records will carry a version, an absolute expiration timestamp, and, for deletion, a tombstone. Absolute expiry gives each replica the same intended deadline, but assumes reasonably synchronized clocks. A client timeout after a successful write can leave the client uncertain whether a retry duplicates an already accepted operation; operation IDs and idempotency remain open design work.

Reads route to the current primary. The coordinator uses one-second heartbeats, marks a node `SUSPECT` after three missed heartbeats, and `FAILED` after five. Nodes also have `HEALTHY` and `RECOVERING` states. To prevent an outdated owner from accepting writes after a membership change, write authority requires both a current topology lease and a monotonically increasing fencing epoch/topology version. A topology version alone is insufficient because an old node may not yet know a newer version exists.

Version 1 has one coordinator. It is deliberately a single point of failure for cluster management: if unavailable, it cannot issue membership changes, failovers, or rebalancing decisions. Restarting the coordinator and recovering its authoritative topology state need further investigation.

Recovery and rebalancing are conservative. A restarted node enters `RECOVERING` and cannot serve traffic until required range synchronization completes. Topology changes are serialized, and writes pause for affected ranges during migration. This simplifies correctness but reduces write availability. The hash ring initially assigns 64 virtual nodes to each physical node; this is an experimental starting point, not a proven optimum.

## Development Roadmap

1. Define API contracts, data semantics, and cluster invariants.
2. Implement and unit-test a single cache node with expiration behavior.
3. Implement deterministic consistent hashing and virtual-node assignment.
4. Add coordinator-backed membership and request routing.
5. Add primary-led synchronous replication, record metadata, and primary reads.
6. Add lease- and fencing-protected health monitoring, failure handling, and serialized rebalancing.
7. Add recovery synchronization and observability.
8. Run fault-injection, integration, and performance characterization tests before making operational claims.

## Testing Strategy

We will test behavior before making reliability or performance claims:

- Unit tests for key operations, expiration, hashing, replica selection, and membership transitions.
- Contract and integration tests for the public API and communication between coordinator and nodes.
- Deterministic multi-node tests covering joins, graceful leaves, process failures, partitions, and recovery.
- Data-placement and invariant checks during rebalancing, including replica-count expectations.
- Fault-injection tests that delay, drop, or duplicate network messages.
- Load and soak tests used to characterize behavior, not to predeclare benchmark results.

Major Version 1 decisions will have direct tests: acknowledged writes must survive a primary failure; stale owners must be fenced from writes; heartbeats must produce the documented state transitions; recovery must not expose data before synchronization; and a migration must preserve records while pausing only its affected write range.

## Open Questions and Future Improvements

- Define operation IDs and idempotency behavior for ambiguous client retries after timeouts.
- Define version generation after a replica is promoted to primary.
- Define when tombstones can be safely removed without permitting stale-value resurrection.
- Define the transfer protocol and retry behavior when recovery synchronization is interrupted.
- Define coordinator restart, persistence, and authoritative topology-recovery behavior.
- Measure whether 64 virtual nodes provide acceptable distribution and movement for local clusters.
- Consider replicated coordination, concurrent migration, quorum replication, and richer observability only after Version 1 behavior is verified.

See [docs/architecture.md](docs/architecture.md) for the proposed component model and [docs/design-decisions.md](docs/design-decisions.md) for decisions that remain open.
