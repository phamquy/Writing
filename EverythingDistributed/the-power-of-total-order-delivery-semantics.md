# The Power of Total Order: Delivery Semantics

In messaging systems and distributed logs (like Kafka, Pulsar, or a simple distinct WAL), **delivery semantics** define the contract between the producer, the system, and the consumer regarding message durability and processing guarantees.

When we have a primitive that guarantees **Total Order** within a partition or shard, we can systematically achieve three distinct levels of delivery guarantees: **At-Most-Once**, **At-Least-Once**, and **Exactly-Once**.

***

### 1. At-Most-Once Delivery

_Definition:_ Messages may be lost, but are never redelivered. If a failure occurs, the message is dropped. _Motto:_ "Better to lose it than to duplicate it."

#### How to achieve it

**Producer Side ("Fire and Forget")**

The producer sends a message to the log but does **not wait for an acknowledgment** (ACK).

* **Mechanism:** `send(msg)` return immediately.
* **Failure Mode:** If the network drops the packet or the leader crashes before writing to disk, the message is lost forever. The producer never retries.

**Consumer Side ("Commit before Process")**

The consumer reads a batch of messages, updates its "read pointer" (offset) to mark them as done, and _then_ starts processing the business logic.

* **Mechanism:**
  1. Fetch Batch `[Msg 100-105]`.
  2. **Commit Offset** `105`.
  3. Process `Msg 100`... Crash!
* **Failure Mode:** Upon restart, the consumer resumes from Offset `106`. Messages `100-105` were technically "delivered" (fetched) but never successfully processed. They are lost.

#### Use Cases

* High-frequency sensor data (IoT) where missing a few data points is acceptable.
* UDP-style log aggregation where speed > completeness.

***

### 2. At-Least-Once Delivery

_Definition:_ Messages are never lost, but may be redelivered. Duplicate processing is possible. _Motto:_ "Better to duplicate it than to lose it."

#### How to achieve it

**Producer Side ("Retry until Ack")**

The producer sends a message and waits for a confirm from the primitive (e.g., Leader + Quorum).

* **Mechanism:**
  1. `send(msg)`
  2. Wait for `ACK`.
  3. If timeout/error: **Retry** indefinitely.
* **Failure Mode:** If the log successfully wrote the message but the `ACK` response was lost on the network, the Producer retries. The log now contains the message **twice**.

**Consumer Side ("Process before Commit")**

The consumer processes the business logic first and only commits the offset after success.

* **Mechanism:**
  1. Fetch Batch `[Msg 100-105]`.
  2. Process `Msg 100`... `Msg 105`. Success.
  3. **Commit Offset** `105`.
* **Failure Mode:**
  * Process `Msg 100`... Success.
  * Process `Msg 101`... Crash!
  * (Offset 105 was never committed).
  * **Restart:** Consumer resumes from Offset `100`. `Msg 100` is processed **again**.

#### Use Cases

* Most standard applications (Payment processing, notifications) where data loss is unacceptable.
* Requires downstream systems to be **Idempotent** to handle the duplicates safely.

***

### 3. Exactly-Once Delivery (Processing)

_Definition:_ Each message is processed effectively once. Neither lost nor duplicated in the final state. _Note:_ In distributed systems, this is technically "Exactly-Once _Processing_ semantics." You can't prevent the network from sending a packet twice, but you can prevent the state from changing twice.

#### How to achieve it

**Method A: Idempotency (The Dedupe Table)**

This relies on the Total Order Log providing unique keys or offsets.

* **Mechanism:**
  1. Consumer reads `Msg 105`.
  2. **Atomic Transaction:**
     * Calculate State Change ($Balance = Balance + 10$).
     * Insert `MsgID_105` into a `ProcessedMessages` table.
     * (Constraint: Fail if `MsgID_105` exists).
  3. Commit.
* **On Retry:** If the consumer crashes and re-reads `Msg 105`, the database rejects the duplicate `MsgID`.

**Method B: Transactional Producer-Consumer (Kafka Style)**

This allows "Atomic Read-Process-Write" cycles where the output of a consumer is a new message to another log topic.

* **Mechanism:**
  1. Producer generates a `TransactionID`.
  2. Writes are marked as `Pending` in the log.
  3. A `COMMIT` marker is written to the log at the end.
  4. **Consumers** define isolation level `read_committed`. They buffer messages and only expose them to the application once the `COMMIT` marker is seen in the log.

**Method C: Idempotent Producer (Sequence Numbers)**

To prevent duplicates at the _ingestion_ level (Writer -> Log):

* **Mechanism:**
  1. Producer creates a Session ID (`PID`) and a monotonically increasing Sequence Number (`Seq=0`).
  2. Send `(PID, Seq=0)`. Log appends it.
  3. Producer retries `(PID, Seq=0)` due to network glitch.
  4. **Log Logic:** The Log sees `Seq=0` is <= the last stored `Seq` for that `PID`. It **acks** the request but **does not append** the duplicate.

#### Use Cases

* Financial transactions (Bank ledgers).
* Ad counting/Billing pipelines.
* Any system where double-counting is a critical bug.

***

#### Summary Table

| Semantic          | Message Loss? | Duplicates? | Throughput | Implementation Complexity            |
| ----------------- | ------------- | ----------- | ---------- | ------------------------------------ |
| **At-Most-Once**  | Yes           | No          | Highest    | Low (Fire & forget)                  |
| **At-Least-Once** | No            | Yes         | High       | Medium (Retries + Offset management) |
| **Exactly-Once**  | No            | No          | Medium     | High (Idempotency / Transactions)    |
