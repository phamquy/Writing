# The Power of Total Order: Achieving Consistency Levels with a Global Log

In distributed systems, achieving consistency is often a trade-off between correctness and performance. However, having a primitive that guarantees **Total Order**—such as a specific Kafka partition, a Raft/Paxos log, or a single-threaded sequencer—significantly simplifies the implementation of various consistency models.

This article explores how we can build [different consistency guarantees](distributed-system-consistency-a-comprehensive-primer.md), ranging from Strict (Linearizability) to Eventual Consistency, leveraging an underlying **Totally Ordered Log**.

#### The Primitive: The Log

Assume we have a system where:

1. All writes are appended to a global, immutable log.
2. Each entry gets a monotonically increasing request ID or **log index** (`0, 1, 2, ...`).
3. All nodes (replicas) consume this log in the exact same order to update their state (State Machine Replication).

***

### 1. Strict Consistency (Linearizability)

_Definition:_ The strongest guarantee. Every read effectively returns the absolute latest write. The system behaves as if there is only one copy of the data.

**How to achieve it:** To achieve Linearizability, time must not "go backward" or "lag" even for a millisecond between the write completing and a subsequent read starting _anywhere_ in the system.

* **Write Path:** Append the write to the log. Wait for the log to commit (replicate to quorum).
* **Read Path (Linearizable Read):**
  1. **Option A (Read-through-Log):** Treat the Read as a write. Append a "Read Request" to the log. When the state machine processes that entry, return the current value. This guarantees the read happens at a specific point in the total order.
  2. **Option B (ReadIndex):** The node receiving the read asks the Leader for the current committed log index $$Index_{latest}$$. The node then waits until its local state machine has applied up to $$Index_{latest}$$ before returning the value.

***

### 2. Sequential Consistency

_Definition:_ Operations appear to take place in some total order, and that order is consistent with the order of operations from each individual client.

**How to achieve it:** This is the "natural" state of a log-based system.

* **Mechanism:** All writes go to the log and are assigned a sequence. All replicas apply writes in that exact log order.
* **The behavior:**
  * If Client A writes $$X=1$$ then $$X=2$$ all replicas see 1 then 2.
  * Replicas can lag. Replica 1 might be at index 100, Replica 2 at index 90.
  * If a user reads from Replica 2, they see "old" data, but they see a _valid prefix_ of the history. They never see updates out of order (e.g., seeing write #100 before write #99).

***

### 3. Causal Consistency

_Definition:_ If operation A "causes" operation B (e.g., B is a reply to A), then every node that sees B must also see A. Concurrent operations can be seen in any order.

**How to achieve it:** If all data resides in the single totally ordered partition, Causal Consistency is automatically achieved (because Sequential implies Causal). However, if the system is sharded (multiple logs):

* **Mechanism:**
  1. When a Client reads a value derived from Log index  $$T_1$$, it gets a "dependency token" (e.g., Vector Clock or simply $$T_{deps} = {LogA: 10, LogB: 5}$$.
  2. When the Client writes new data, it attaches $T\_{deps}$.
  3. The system ensures this new write is not visible to others until the dependencies in $$T_{deps}$$ are satisfied.

***

### 4. Read-Your-Writes

_Definition:_ A client is guaranteed to see the effects of its _own_ previous writes.

**How to achieve it:**

* **State:** The Client (or the session gateway) tracks the `LastWriteIndex` it successfully committed.
* **Mechanism:**
  1. Client writes data, receives ack with Log Index $I\_{write}$.
  2. Client sets `ClientSession.LastSeenIndex = I_{write}`.
  3. Client sends a Read request including `LastSeenIndex`.
  4. Replica receives request. Checks its own `AppliedIndex`.
  5. **If** `Replica.AppliedIndex >= Client.LastSeenIndex`: Return data immediately.
  6. **Else**: Block (wait) until the replica catches up, or redirect the client to a fresher replica.

***

### 5. Monotonic Reads

_Definition:_ If a user reads a data version `V` ,they will never see a version older than `V` in subsequent reads. "Time doesn't move backward."

**How to achieve it:** Commonly used when users are typically sticky to a replica, but might switch (e.g., mobile roaming).

* **State:** The Client tracks the `LastReadIndex`.
* **Mechanism:**
  1. Client performs a Read, Replica responds with data at Log Index 100.
  2. Client updates `ClientSession.LastReadIndex = 100`.
  3. Client connects to a _different_ Replica (e.g., after a load balancer switch) that is lagging at Index 90.
  4. New Replica sees `LastReadIndex = 100` > `LocalIndex = 90`.
  5. Replica **waits** until it consumes up to index 100 before answering.

***

### 6. Monotonic Writes

_Definition:_ Writes from the same client are applied in the order they were issued.

**How to achieve it:** With a single total order log, this is largely about the Producer implementation.

* **Mechanism:**
  1. The Client (Producer) sends Write A.
  2. The Client waits for the ack of Write A before sending Write B.
  3. **Optimization (Pipelining):** The Client assigns sequence numbers ( $$Seq_1, Seq_2$$) to the batches. The Log Leader ensures that $Seq\_2$ is only written after $$Seq_1$$ is successfully written. Kafka's `enable.idempotence=true` does exactly this to prevent reordering during retries.

***

### 7. Bounded Staleness

_Definition:_ Reads may lag behind the latest data, but only by a specific quantity (time or number of versions).

**How to achieve it:**

* **Mechanism:**
  1. The Leader knows the precise `HighWaterMark` (latest committed index).
  2. Replicas gossip or fetch this HighWaterMark.
  3. When a Client reads from a Replica, the Replica checks: `Lag = HighWaterMark - LocalAppliedIndex`
  4. **Policy:**
     * If `Lag < K` (max allowed staleness): Serve the read.
     * If `Lag > K`: Reject the read or block until caught up.

***

### 8. Eventual Consistency

_Definition:_ If no new updates are made, eventually all replicas will converge to the same state.

**How to achieve it:**

* **Mechanism:**
  * Replicas consume the Total Order Log at their own pace.
  * No blocking on reads. No checks against client tokens.
  * Simply: `return local_state.get(key)`.
  * Since the log is immutable and ordered, "Convergence" is mathematically guaranteed. They typically won't diverge (conflict), they just vary in _time_.

***

#### Summary Table

| consistency Model         | Required Client Metadata | Replica Check Needed?                | Performance Cost              |
| ------------------------- | ------------------------ | ------------------------------------ | ----------------------------- |
| **Strict (Linearizable)** | None                     | **Yes** (Consensus/Lease check)      | High (Round-trips to leader)  |
| **Sequential**            | None                     | No (Log order implicitly handles it) | Low                           |
| **Read-Your-Writes**      | `LastWriteIndex`         | Yes (Wait for index)                 | Medium (Wait only if lagging) |
| **Monotonic Reads**       | `LastReadIndex`          | Yes (Wait for index)                 | Medium (Wait only if lagging) |
| **Eventual**              | None                     | No                                   | Lowest                        |
