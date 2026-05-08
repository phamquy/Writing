---
icon: cloud-binary
---

# Data Synchronization – Ensure consistency between two services.

#### Mechanisms to Ensure Consistency Between Two Services

**1. Event-Driven Synchronization**

* Use an event-driven architecture where changes in one service trigger events consumed by the other service.
* Example: Publish-Subscribe model using message brokers like Kafka or RabbitMQ.

**2. Periodic Data Sync**

* Schedule periodic synchronization tasks to reconcile data between services.
* Example: Compare and update records every hour.

**3. Two-Way Sync with Conflict Resolution**

* Implement bi-directional synchronization with conflict resolution strategies (e.g., last-write-wins or custom rules).
* Example: Sync user profiles between a mobile app and a web app.

**4. API-Based Sync**

* Use APIs to push or pull updates between services in real-time or batch mode.
* Example: Service A calls Service B's API to update data after a change.

**5. Distributed Transactions**

* Use distributed transaction protocols like Two-Phase Commit or Saga Pattern to ensure atomicity across services.
* Example: Ensure both services commit or rollback changes together.

**6. Data Validation and Checks**

* Periodically validate data consistency using checksums or hash comparisons.
* Example: Compare database snapshots or logs.

**7. Shared Data Store**

* Use a shared database or distributed storage system to maintain a single source of truth.
* Example: Both services access the same database.

***

#### Levels of Consistency in Distributed Systems

**1. Strong Consistency**

* Guarantees all replicas see the same data at the same time.
* Example: Distributed databases using Paxos or Raft protocols.

**2. Eventual Consistency**

* Ensures all replicas converge to the same state eventually, but not immediately.
* Example: DNS systems or NoSQL databases like Cassandra.

**3. Causal Consistency**

* Preserves the order of causally related operations but does not guarantee global ordering.
* Example: Collaborative editing tools.

**4. Read-Your-Writes Consistency**

* Ensures a user sees their own updates immediately.
* Example: User profile updates in web applications.

**5. Monotonic Read Consistency**

* Guarantees once a user reads a value, they will not see an older value in subsequent reads.
* Example: Versioned data in distributed systems.

**6. Linearizability**

* Ensures operations appear to occur in a single, sequential order across all replicas.
* Example: Atomic operations in distributed key-value stores.

**7. Sequential Consistency**

* Guarantees operations appear in the same order across all replicas, but not necessarily in real-time.
* Example: Shared memory systems.

**8. Bounded Staleness**

* Ensures replicas lag behind the primary by a bounded amount of time or updates.
* Example: Content delivery networks (CDNs).

**9. Quorum Consistency**

* Requires a majority of replicas to agree on the state before committing changes.
* Example: Distributed databases using quorum reads/writes.

**10. Partial Consistency**

* Guarantees consistency for a subset of operations or data, while others may remain inconsistent.
* Example: Systems with relaxed constraints for performance optimization.
