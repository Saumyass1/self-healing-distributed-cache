# Proposed Design Decisions

## Status

All entries are proposals for investigation, not final choices or implemented behavior.

## Runtime and Service Interface

**Problem:** The project needs an accessible implementation environment and a way for clients and cluster members to communicate.

**Proposed approach:** Use TypeScript on Node.js and begin with HTTP service interfaces.

**Why consider it:** TypeScript offers a low-friction environment for a learning-oriented distributed-systems project, while HTTP is inspectable with common tools and straightforward to test.

**Alternatives:** Go with gRPC or HTTP; Java with a mature distributed-systems ecosystem; Rust for tighter resource control; Node.js with a binary RPC protocol.

**Trade-offs:** TypeScript and HTTP favor iteration and transparency but add runtime overhead and may be less suitable for high-throughput production workloads. A binary protocol improves efficiency at the cost of observability and implementation complexity.

**Questions to investigate:** Which framework, validation library, timeouts, and error format fit the API? Should external clients and node-to-node traffic use different protocols?

## Key Placement

**Problem:** Keys must be distributed predictably while minimizing movement when cluster membership changes.

**Proposed approach:** Use consistent hashing with multiple virtual nodes per physical cache node.

**Why consider it:** It supports deterministic routing and usually moves only affected hash ranges as nodes join or leave. Virtual nodes can smooth uneven distribution.

**Alternatives:** Modulo hashing; rendezvous hashing; fixed partition tables; a coordinator-selected routing map.

**Trade-offs:** A hash ring is conceptually manageable but introduces ring-version coordination and virtual-node tuning. Rendezvous hashing can simplify membership calculations but has different computational and replication characteristics.

**Questions to investigate:** Which stable hash function should be used? How many virtual nodes are justified by observed balance? How is the ring serialized and versioned?

## Coordinator Role and Availability

**Problem:** Participants need an authoritative membership and topology view.

**Proposed approach:** Start with a dedicated coordinator that manages membership and publishes versioned topology snapshots.

**Why consider it:** Centralizing the initial control plane makes membership transitions and their observability easier to understand.

**Alternatives:** Gossip-based membership; an external coordination system; a replicated consensus-backed coordinator; static membership for the earliest prototype.

**Trade-offs:** A single coordinator simplifies the initial build but is a control-plane availability risk. Distributed membership improves resilience but adds disagreement, convergence, and split-brain concerns.

**Questions to investigate:** Does the coordinator need replication before failure automation is introduced? What can nodes safely do during coordinator unavailability? How are topology updates authenticated and ordered?

## Replication Consistency

**Problem:** Replicas improve availability, but reads and writes can observe different versions during delay, failure, or rebalancing.

**Proposed approach:** Begin by evaluating a configurable replication factor with primary-led replication, while postponing the acknowledgement and read-consistency policy.

**Why consider it:** Primary-led replication gives a clear ownership path and makes the first replication flow easier to reason about.

**Alternatives:** Synchronous replication to all replicas; quorum reads and writes; asynchronous replication; client-side fan-out; no replication in the first executable milestone.

**Trade-offs:** Stronger acknowledgement requirements reduce stale reads and data loss windows but increase latency and reduce availability during failures. Asynchronous replication is fast but can lose recently acknowledged cache writes. Quorums require versioning and conflict handling.

**Questions to investigate:** What consistency promise should `GET` after `SET` have? What acknowledgement count is required for success? How are concurrent writes ordered or resolved? How are deletes, TTLs, and stale replicas handled? This is an open architectural decision.

## Failure Detection Thresholds

**Problem:** The cluster must distinguish a failed node from a slow node or a temporary network partition.

**Proposed approach:** Evaluate heartbeat-based detection with a suspicion state before declaring a node failed.

**Why consider it:** It is observable and understandable for an initial coordinator-based model, and a suspicion period can reduce unnecessary rebalancing.

**Alternatives:** Active probes only; phi-accrual detection; gossip-based failure detection; immediate removal after a missed heartbeat.

**Trade-offs:** Aggressive thresholds react faster but risk false positives and churn. Conservative thresholds preserve stability but increase time spent routing around a genuinely failed node. No detector can reliably distinguish all failures from partitions.

**Questions to investigate:** What heartbeat interval, timeout, missed-heartbeat count, and suspicion duration are appropriate? How are partitions represented? Who can confirm failure? What metrics validate false-positive and detection-delay behavior? These thresholds are open design questions.

## Recovered-Node Synchronization

**Problem:** A previously failed node can return with missing or stale memory state and must rejoin without corrupting placement or serving stale values.

**Proposed approach:** Treat recovery as a staged process: register, receive a current topology, synchronize assigned ranges from active owners, verify completion, then become eligible to serve its assigned traffic.

**Why consider it:** Separating reachability from serving eligibility prevents a restarted node from immediately advertising incomplete or obsolete data.

**Alternatives:** Discard all local state and warm from misses; immediately rejoin and repair lazily on reads; use anti-entropy continuously; persist cache data locally between restarts.

**Trade-offs:** Eager synchronization improves readiness and predictability but consumes network and memory capacity. Lazy repair reduces recovery time but permits more misses or stale reads. Persistence complicates a cache that otherwise treats data as ephemeral.

**Questions to investigate:** Which owner supplies each range during concurrent rebalances? Is local recovered state discarded, reconciled, or version-checked? How is synchronization made idempotent and resumable? When is the node safe to receive primary traffic? This is an open architectural decision.

## Observability

**Problem:** Self-healing behavior cannot be trusted or debugged without visibility into topology, failures, and data movement.

**Proposed approach:** Expose structured logs and Prometheus-compatible metrics from all control-plane and data-plane services, with a dashboard added after metric semantics stabilize.

**Why consider it:** Metrics make recovery behavior measurable and support automated tests that assert state transitions.

**Alternatives:** Logs only; OpenTelemetry-first tracing; a hosted observability service; custom event storage.

**Trade-offs:** Metrics are efficient for aggregate behavior but may omit per-request history. Tracing improves causal diagnosis but adds instrumentation and collection complexity.

**Questions to investigate:** Which labels avoid excessive cardinality? Which signals define rebalance health? What retention and alerting are appropriate for a local development cluster?
