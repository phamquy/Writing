# Linearizability vs Serializability vs Total Order

#### 🧩 1. Linearizability

**Definition:**\
A real‑time consistency model for **single‑object operations**. Each operation appears to take effect at a single instant between invocation and response.

**Properties**

* Respects **real‑time order**
* Applies to **individual reads/writes**
* Strongest guarantee for single‑object semantics
* If A completes before B starts → A must appear before B

**Use cases**

* Strongly consistent key‑value stores
* Coordination primitives (locks, leader election)

***

#### 🧩 2. Serializability

**Definition:**\
A correctness condition for **transactions**. The final result must be equivalent to _some_ sequential execution of all transactions.

**Properties**

* No real‑time constraint
* Ensures isolation across multi‑operation, multi‑object transactions
* Allows reordering as long as the outcome matches a legal serial schedule

**Strict Serializability**

* Serializability **+ real‑time order**
* Equivalent to linearizability but for transactions

**Use cases**

* Databases
* Distributed transactions
* ACID systems

***

#### 🧩 3. Total Order

**Definition:**\
A guarantee that **all nodes observe operations/messages in the exact same order**.

**Properties**

* Global sequence shared by all replicas
* No inherent real‑time requirement
* Typically implemented via total‑order broadcast or a sequencer

**Use cases**

* Replicated state machines
* Consensus protocols (Paxos, Raft)
* Log replication

***

#### 🔍 Summary Table

| Model                      | Scope             | Real‑time?     | Guarantees                         | Typical Systems       |
| -------------------------- | ----------------- | -------------- | ---------------------------------- | --------------------- |
| **Linearizability**        | Single operations | ✔️ Yes         | Real‑time‑consistent atomic ops    | Etcd, Zookeeper       |
| **Serializability**        | Transactions      | ❌ No           | Equivalent to some serial schedule | Databases             |
| **Strict Serializability** | Transactions      | ✔️ Yes         | Serializability + real‑time        | Spanner, FoundationDB |
| **Total Order**            | Messages/events   | ❌ Not required | Same global order everywhere       | Paxos, Raft           |

***

#### 🧠 Relationship Summary

* **Linearizability** ⇒ total order that respects real time
* **Serializability** ⇒ total order of transactions (no real‑time constraint
