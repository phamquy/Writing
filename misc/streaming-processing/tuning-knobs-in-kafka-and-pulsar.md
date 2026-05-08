---
icon: flask-gear
---

# Tuning Knobs in Kafka & Pulsar

## ⚙️ Key Tuning Knobs in Kafka & Pulsar

### 1. Partition Count

* **What it does:** Splits a topic into multiple partitions for parallelism. Each partition is an ordered log.
* **Increase it:**
  * ✅ Higher throughput (more parallelism, more consumers can process in parallel).
  * ❌ More overhead (brokers, ZooKeeper/metadata load, open file handles, rebalancing cost).
* **Decrease it:**
  * ✅ Lower overhead.
  * ❌ Reduced parallelism, possible consumer bottlenecks.

***

### 2. Producer Batch Size & Linger

#### Kafka

* `batch.size` (bytes), `linger.ms` (delay before sending).

#### Pulsar

* `batchingMaxMessages`, `batchingMaxPublishDelay`.
* **What it does:** Groups multiple messages into one request before sending to broker.
* **Increase it:**
  * ✅ Better throughput, reduced network overhead.
  * ❌ Higher end-to-end latency (messages wait in buffer).
* **Decrease it:**
  * ✅ Lower latency (messages sent quickly).
  * ❌ Worse throughput (more requests, higher CPU/network overhead).

***

### 3. Replication Factor & Acknowledgments

#### Kafka

* `default.replication.factor`, `acks` (0, 1, all).

#### Pulsar

* `replicationClusters`, `sendTimeoutMs`.
* **What it does:** Controls durability of messages (how many copies are stored and when producer considers write "done").
* **Increase it:**
  * ✅ Higher durability (data survives broker failures).
  * ❌ Slower writes, more storage + network cost.
* **Decrease it:**
  * ✅ Faster writes, less storage overhead.
  * ❌ Higher risk of data loss.

***

### 4. Consumer Prefetch / Fetch Size

#### Kafka

* `fetch.min.bytes`, `fetch.max.wait.ms`.

#### Pulsar

* `receiverQueueSize`.
* **What it does:** Controls how many messages are pulled and buffered at consumer side.
* **Increase it:**
  * ✅ Higher throughput (fewer round trips, better batching).
  * ❌ Higher memory usage, possible message "burst" load, longer processing latency for first message in batch.
* **Decrease it:**
  * ✅ Lower memory footprint, faster responsiveness to new data.
  * ❌ Lower throughput due to more broker requests.

***

### 5. Retention Policy

#### Kafka

* `log.retention.hours`, `log.segment.bytes`.

#### Pulsar

* `retentionPolicies`, `managedLedgerMaxEntriesPerLedger`, `offloadPolicies`.
* **What it does:** Determines how long messages stay in the log (and/or cold storage).
* **Increase it:**
  * ✅ Longer replay window (consumers can re-read old data, supports backfill & analytics).
  * ❌ More storage cost, more compaction/cleanup load.
* **Decrease it:**
  * ✅ Lower storage footprint, less broker pressure.
  * ❌ Reduced replay window, possible data loss if consumers lag.

***

### 6. Compression

#### Kafka

* `compression.type` (none, gzip, snappy, lz4, zstd).

#### Pulsar

* `compressionType` (same algorithms).
* **What it does:** Shrinks message size before sending over network or writing to disk.
* **Increase (stronger compression):**
  * ✅ Lower network/disk usage, better throughput when network-bound.
  * ❌ Higher CPU usage, possible latency increase.
* **Decrease (no compression):**
  * ✅ Lower CPU usage, lower producer latency.
  * ❌ Larger data size, more I/O bottleneck.

***

### 7. Consumer Rebalancing / Subscription Model

#### Kafka

* Consumer groups → rebalancing partitions among consumers.

#### Pulsar

* Subscription type: `Exclusive`, `Failover`, `Shared`, `Key_Shared`.
* **What it does:** Defines how consumers divide work and ensure ordering.
* **Shared / Group model (increase concurrency):**
  * ✅ Parallel processing, load balancing.
  * ❌ Possible out-of-order delivery.
* **Exclusive / Single consumer (decrease concurrency):**
  * ✅ Strong ordering guarantee.
  * ❌ Single consumer bottleneck.

***

### 🎯 Summary of Effects

* **Increase knob value:** More throughput, more durability, more storage/CPU cost, higher latency.
* **Decrease knob value:** Lower resource usage, lower latency, but less throughput, durability, or replayability.

## 📊 Key Performance Metrics in Kafka & Pulsar

### 1. Throughput

* **Definition:** Number of messages (or bytes) the system can process per second.
* **Why important:** High throughput = system can handle more events per unit time.
* **Knobs affecting it:**
  * Producer batching (`batch.size`, `linger.ms`, `batchingMaxMessages`)
  * Compression (`compression.type`, `compressionType`)
  * Partition count (`num.partitions`, `partitions`)
  * Consumer prefetch size (`fetch.min.bytes`, `receiverQueueSize`)
* **Trade-offs:** Increasing throughput often increases **latency** and **resource usage**.

***

### 2. Latency

* **Definition:** Time from when a message is produced until it is consumed.
* **Why important:** Low latency = real-time responsiveness.
* **Knobs affecting it:**
  * Producer linger (`linger.ms`, `batchingMaxPublishDelay`)
  * Consumer fetch wait (`fetch.max.wait.ms`)
  * Replication acks (`acks`, `sendTimeoutMs`)
* **Trade-offs:** Reducing latency often reduces **throughput** (fewer batches, more network trips).

***

### 3. Durability / Reliability

* **Definition:** Probability that a message will not be lost, even if brokers crash.
* **Why important:** Critical for financial, audit, and mission-critical systems.
* **Knobs affecting it:**
  * Replication factor (`default.replication.factor`, `replicationClusters`)
  * Acknowledgments (`acks=all`, Pulsar `WaitForAll`)
  * Retention policies (longer retention = safer replay)
* **Trade-offs:** Higher durability = slower writes, more storage and network overhead.

***

### 4. Scalability / Parallelism

* **Definition:** Ability to add more producers, brokers, or consumers to scale load.
* **Why important:** Determines how well system grows with traffic.
* **Knobs affecting it:**
  * Partition count (Kafka `num.partitions`, Pulsar partitioned topics)
  * Consumer group/subscription type
* **Trade-offs:** Too many partitions = metadata overhead, controller load.

***

### 5. Storage Efficiency

* **Definition:** Disk and memory footprint per message.
* **Why important:** Impacts cost and how long history can be retained.
* **Knobs affecting it:**
  * Compression type (gzip/zstd/lz4/snappy)
  * Retention policies (`log.retention.hours`, `retentionPolicies`)
  * Segment / ledger sizing (`log.segment.bytes`, `managedLedgerMaxEntriesPerLedger`)
* **Trade-offs:** Smaller storage footprint = more CPU for compression, or risk of data loss if retention is too short.

***

### 6. Consumer Lag

* **Definition:** Difference between latest message offset and consumer’s committed offset.
* **Why important:** Shows if consumers can keep up with producers.
* **Knobs affecting it:**
  * Consumer fetch/prefetch size (`fetch.min.bytes`, `receiverQueueSize`)
  * Consumer concurrency (group size, subscription type)
* **Trade-offs:** Reducing lag may require more partitions, more consumers, or bigger buffers.

***

## 🎯 Putting It Together

* **Throughput vs Latency** → Batch size & linger.
* **Durability vs Performance** → Replication & acks.
* **Scalability vs Overhead** → Partition count & consumer group size.
* **Storage vs Replay** → Retention & compression.

### Every knob is about balancing these **six metrics**:<br>

➡️ _Throughput, Latency, Durability, Scalability, Storage, Lag_.

