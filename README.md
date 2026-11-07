# Self-Healing Distributed Cache System

## Project Status

This repository currently contains planning documentation only. No cache, coordinator, dashboard, infrastructure, or automated tests have been implemented.

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

## Development Roadmap

1. Define API contracts, data semantics, and cluster invariants.
2. Implement and unit-test a single cache node with expiration behavior.
3. Implement deterministic consistent hashing and virtual-node assignment.
4. Add coordinator-backed membership and request routing.
5. Add replication and explicitly chosen consistency semantics.
6. Add health monitoring, failure handling, and rebalancing.
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

See [docs/architecture.md](docs/architecture.md) for the proposed component model and [docs/design-decisions.md](docs/design-decisions.md) for decisions that remain open.
