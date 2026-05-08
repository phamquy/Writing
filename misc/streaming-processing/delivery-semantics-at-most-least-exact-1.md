---
icon: timeline-arrow
---

# Delivery semantics - at most/least, exact 1

## Why "At Most Once" and "At Least Once" Often Ignore Producer Side

When people describe messaging semantics (like _at most once_ or _at least once_), they’re usually talking about the **consumer’s perspective** — i.e., what the application that processes messages will observe.

Producer-side guarantees (whether a message successfully reached the broker) are important, but in practice:

* **At most once** and **at least once** are defined at the **delivery from broker → consumer** stage.
* The producer side is often assumed to have _some level of reliability_ (e.g., retries, acknowledgments), otherwise no delivery guarantee can be discussed at all.

***

### 1️⃣ At Most Once

* The consumer may acknowledge/commit **before** processing.
* If the consumer crashes after acknowledging, the message is lost.
* Even if the producer sent reliably, the consumer’s ack timing makes it _at most once_.
* → Producer doesn’t change this outcome.

***

### 2️⃣ At Least Once

* The consumer only acknowledges **after** processing.
* If the consumer crashes before acknowledging, the broker will redeliver the same message.
* This may cause duplicates, but guarantees no loss.
* → Producer reliability helps, but the definition of _at least once_ is entirely about consumer ack timing.

***

### 3️⃣ Exactly Once

* Only here do both sides matter.
* Requires:
  * **Producer idempotence** (avoid duplicates on retry).
  * **Consumer transactions** (process + commit atomically).
* Both producer and consumer must cooperate to achieve this.

***

#### 🔑 Key Insight

* **At most once / at least once** are primarily about the **consumer–broker interaction**.
* **Exactly once** requires coordination across both producer and consumer.



Delivery semantics depend on both producer-side guarantees (producer → broker) and consumer-side guarantees (broker → consumer).

***

### 1️⃣ Producer → Broker Guarantees

| Producer Config          | Kafka Example                          | Pulsar Example       | Semantics                                                    |
| ------------------------ | -------------------------------------- | -------------------- | ------------------------------------------------------------ |
| `acks=0`                 | fire-and-forget                        | send without waiting | **At most once** (loss possible)                             |
| `acks=1`                 | wait for leader only                   | wait for broker      | **At least once** (loss unlikely, dup possible with retries) |
| `acks=all` + idempotence | wait for all replicas + sequence dedup | dedup enabled        | **Exactly once** (no loss, no dup)                           |

***

### 2️⃣ Broker → Consumer Guarantees

| Consumer Handling                | Kafka Example          | Pulsar Example         | Semantics                                 |
| -------------------------------- | ---------------------- | ---------------------- | ----------------------------------------- |
| Commit/ack **before** processing | commit offset early    | ack before logic       | **At most once** (loss possible)          |
| Commit/ack **after** processing  | commit offset late     | ack after logic        | **At least once** (no loss, dup possible) |
| Transactions / atomic commits    | transactional consumer | Pulsar transaction API | **Exactly once** (no loss, no dup)        |

***

### 3️⃣ Combined Matrix (Producer × Consumer)

| Producer → Broker                         | Consumer → App            | End-to-End Guarantee                                                  |
| ----------------------------------------- | ------------------------- | --------------------------------------------------------------------- |
| **At most once** (acks=0)                 | At most once (ack before) | ❌ Worst: data may vanish at producer or consumer                      |
| **At most once**                          | At least once (ack after) | ⚠️ Still at most once: lost msgs at producer cannot be recovered      |
| **At least once** (acks=1)                | At most once              | ⚠️ At most once overall (loss possible if consumer crashes)           |
| **At least once**                         | At least once             | ✅ At least once overall (possible duplicates)                         |
| **At least once**                         | Exactly once              | ⚠️ At least once overall (producer retries can cause dups)            |
| **Exactly once** (acks=all + idempotence) | At least once             | ⚠️ At least once overall (consumer may redeliver)                     |
| **Exactly once**                          | Exactly once              | ✅ End-to-end exactly once (ideal, but needs strict config + overhead) |

***

### 🔑 Key Takeaways

* **End-to-end semantics = weakest link**.
  * If either producer or consumer is _at most once_, the whole pipeline is at most once.
* **Exactly-once** requires:
  1. Producer is idempotent (dedup at broker).
  2. Consumer commits offsets/messages transactionally.
