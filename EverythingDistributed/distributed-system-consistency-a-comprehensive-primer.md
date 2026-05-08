# Distributed System Consistency: A Comprehensive Primer

### Overview

Consistency in distributed systems defines the guarantees about what values read operations can return after write operations. The CAP theorem tells us we cannot have all three of Consistency, Availability, and Partition tolerance simultaneously—we must make trade-offs.

This primer covers consistency models from **strictest to loosest**, explaining what each means, how to achieve it, and real-world examples.

***

### Consistency Spectrum

```
STRICT ◄─────────────────────────────────────────────────────────► LOOSE
  │                                                                    │
  ├─ Strict (Linearizability)                                          │
  ├─ Sequential Consistency                                            │
  ├─ Causal Consistency                                                │
  ├─ Read-Your-Writes                                                  │
  ├─ Monotonic Reads                                                   │
  ├─ Monotonic Writes                                                  │
  ├─ Bounded Staleness                                                 │
  └─ Eventual Consistency ─────────────────────────────────────────────┘
```

***

### 1. Strict Consistency (Linearizability)

#### What It Is

The **strongest** consistency model. Every operation appears to take effect instantaneously at some point between its invocation and completion. All processes see the same order of operations, and that order respects real-time ordering.

**Key Properties:**

* Operations appear atomic and instantaneous
* Real-time ordering is preserved
* Any read returns the most recent write
* All nodes agree on the order of all operations

#### How to Achieve It

```
┌─────────────────────────────────────────────────────────────┐
│                    LINEARIZABILITY                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Techniques:                                                │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ 1. Single Leader with Synchronous Replication       │    │
│  │    - All writes go through one node                 │    │
│  │    - Wait for ALL replicas to acknowledge           │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ 2. Consensus Protocols                              │    │
│  │    - Paxos, Raft, Zab                               │    │
│  │    - Majority quorum must agree                     │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ 3. Distributed Locks / Leases                       │    │
│  │    - Acquire lock before operation                  │    │
│  │    - Release after completion                       │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

Let's unpack each approach:

**Single Leader with Synchronous Replication** is the simplest way to achieve strict consistency—every write must be confirmed as replicated to all replicas before returning success. However, it comes with significant drawbacks:

* The leader becomes a **single point of failure** and a scaling bottleneck.
* When the leader crashes, a new leader must be elected—bringing us back to the consensus problem. Even protocols with deterministic leader selection (like chain replication) still require all nodes to agree on the current leader's state (alive or dead).
* Waiting for **all** replicas to acknowledge increases write latency. Any single replica failure impacts write performance until it's replaced.

**Consensus Algorithms** (Paxos, Raft, Zab) ensure consistency through voting protocols. A write succeeds when a **majority** of nodes agree, making the system more resilient than single-leader setups. These algorithms differ in implementation, but generally either elect a leader to coordinate consensus or operate in leaderless mode where every write requires a quorum vote. Either way, the coordination overhead still impacts write latency.

**Distributed Locks** delegate consensus to an external coordination cluster (etcd, ZooKeeper, Consul). Instead of the data store itself achieving consensus, clients must first acquire a lock from a linearizable lock service before performing writes. The latency shifts from the write operation to lock acquisition—but you're essentially outsourcing the hard consensus problem to a specialized system built for it.

Take a way from here is that linearizablity or strict consistency come at cost of latency and through put. Use it only you have to, like mention in the use case below

**Implementation Pattern (Raft-based):**

```python
class LinearizableKVStore:
    def write(self, key, value):
        # 1. Leader receives write
        # 2. Leader appends to local log
        # 3. Leader replicates to followers
        # 4. Wait for majority acknowledgment
        # 5. Commit and apply to state machine
        # 6. Respond to client
        
    def read(self, key):
        # Option A: Read from leader with lease
        # Option B: Read goes through Raft log (linearizable read)
        # Option C: ReadIndex - verify leader is still leader
```

#### Real-World Examples

* **etcd / Consul**: Distributed key-value stores using Raft
* **ZooKeeper**: Coordination service using Zab protocol
* **Google Spanner**: Globally distributed database with TrueTime
* **CockroachDB**: Distributed SQL with serializable isolation

#### Trade-offs

| Pros                    | Cons                                 |
| ----------------------- | ------------------------------------ |
| Strongest guarantees    | High latency (consensus overhead)    |
| Easiest to reason about | Lower availability during partitions |
| No stale reads          | Limited throughput                   |

#### When to Use

✅ **Use linearizability when:**

* **Distributed locks / leader election**: Only one node should hold a lock or be leader at any time. A stale read could cause split-brain.
* **Unique constraints**: Usernames, email addresses, inventory reservations—where duplicates cause real business harm.
* **Financial transactions**: Account balances, payment processing where double-spend must be prevented.
* **Configuration management**: Feature flags, routing rules that must be consistent across all nodes immediately.
* **Coordination primitives**: Barriers, semaphores, sequence number generation.

❌ **Avoid when:**

* Read-heavy workloads where staleness is acceptable
* Global deployments where cross-region latency is prohibitive
* High-throughput scenarios where consensus becomes a bottleneck
* Data that is naturally append-only or immutable

***

### 2. Sequential Consistency

#### What It Is

All processes see the **same order of operations**, but that order doesn't need to respect real-time. Operations from each process appear in program order, but operations from different processes can be interleaved arbitrarily.

**Key Difference from Linearizability:**

* Linearizability: Real-time order matters
* Sequential: Only program order within each process matters

```
Timeline Reality:
  Process A:  W(x=1)─────────────────R(x)
  Process B:       W(x=2)────R(x)

Linearizable: Process A's R(x) must return 2 (latest write)
Sequential:   Process A's R(x) could return 1 (if we reorder W(x=2) after R(x))
              As long as ALL processes see the same order
```

#### How to Achieve It

1. **Total Order Broadcast**: All nodes receive messages in same order
2. **Lamport Timestamps**: Logical clocks to order events
3. **Single Sequencer**: One node assigns sequence numbers

```
┌────────────┐     ┌────────────┐     ┌────────────┐
│  Client A  │     │ Sequencer  │     │  Client B  │
└─────┬──────┘     └─────┬──────┘     └─────┬──────┘
      │                  │                  │
      │──W(x=1)─────────►│                  │
      │                  │ seq=1            │
      │                  │─────────────────►│──W(x=2)──► │
      │                  │ seq=2            │            │
      │                  │◄─────────────────│            │
      │                  │                  │            │
      │       Broadcast: [seq=1:W(x=1), seq=2:W(x=2)]    │
      │                  │                  │            │
```

**Deep Dive: Lamport Timestamps**

Lamport timestamps provide a simple way to establish a **partial ordering** of events in a distributed system without synchronized physical clocks.

**Core Rules:**

```
┌─────────────────────────────────────────────────────────────┐
│                   LAMPORT CLOCK RULES                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Each process maintains a counter C                         │
│                                                             │
│  Rule 1: Before any local event                             │
│          C = C + 1                                          │
│                                                             │
│  Rule 2: When sending a message                             │
│          C = C + 1                                          │
│          Attach C to the message                            │
│                                                             │
│  Rule 3: When receiving a message with timestamp T          │
│          C = max(C, T) + 1                                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Visual Example:**

```
Process A          Process B          Process C
   │                  │                  │
   │ C=1              │                  │
   ├──────────────────┼──────────────────┤
   │                  │                  │
   │ event            │                  │
   │ C=1              │                  │
   │                  │                  │
   │─────msg(1)──────►│                  │
   │ C=2              │                  │
   │                  │ recv: max(0,1)+1 │
   │                  │ C=2              │
   │                  │                  │
   │                  │ event            │
   │                  │ C=3              │
   │                  │                  │
   │                  │─────msg(3)──────►│
   │                  │ C=4              │
   │                  │                  │ recv: max(0,3)+1
   │                  │                  │ C=4
   │                  │                  │
   │◄────────────────msg(4)──────────────│
   │                                     │ C=5
   │ recv: max(2,4)+1                    │
   │ C=5                                 │
```

**Key Properties:**

| Property                 | Meaning                                                |
| ------------------------ | ------------------------------------------------------ |
| **Causality preserved**  | If event A → B (A happened-before B), then C(A) < C(B) |
| **Partial order only**   | C(A) < C(B) does NOT mean A → B (could be concurrent)  |
| **No wall-clock needed** | Works without synchronized physical time               |
| **Lightweight**          | Just a single integer counter                          |

**The Limitation:** Two concurrent events (no message between them) can have the same timestamp. Solution: add process ID as tie-breaker → `(timestamp, process_id)` for total ordering.

**Vector Clocks (Extension):** For detecting **true concurrency**, each process maintains a vector of counters (one per process). This allows distinguishing between "happened-before" and "concurrent" relationships, unlike Lamport timestamps which only provide partial ordering.

#### Real-World Examples

* **Apache Kafka**: Sequential ordering within partitions
* **Amazon SQS FIFO**: Ordered message delivery
* **TiDB**: Distributed database with sequential consistency option

#### Trade-offs

| Pros                          | Cons                        |
| ----------------------------- | --------------------------- |
| All nodes see same history    | Still requires coordination |
| Easier than linearizability   | Doesn't respect real-time   |
| Good for total ordering needs | Sequencer can be bottleneck |

#### When to Use

✅ **Use sequential consistency when:**

* **Event sourcing / audit logs**: All consumers must see events in the same order to reconstruct state correctly.
* **Message queues**: Order matters for processing (e.g., "create order" before "ship order").
* **Replicated state machines**: All replicas apply commands in same order to stay in sync.
* **Collaborative editing history**: Undo/redo must work consistently across all clients.
* **Database replication logs**: Binlog, WAL consumers need deterministic ordering.

❌ **Avoid when:**

* Real-time ordering matters (use linearizability instead)
* Operations are independent and can be parallelized
* Global scale where a single sequencer becomes a latency/availability bottleneck

***

### 3. Causal Consistency

#### What It Is

Operations that are **causally related** must be seen by all processes in the same order. Concurrent operations (no causal relationship) can be seen in different orders by different processes.

**Causal Relationships:**

* Read-after-write: If B reads what A wrote, B's operations are causally after A's write
* Transitivity: If A → B and B → C, then A → C

```
┌─────────────────────────────────────────────────────────────┐
│                    CAUSAL CONSISTENCY                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  User A posts: "I got the job!"                             │
│       │                                                     │
│       ▼ (causal dependency)                                 │
│  User B comments: "Congratulations!"                        │
│       │                                                     │
│       ▼ (causal dependency)                                 │
│  User A replies: "Thanks!"                                  │
│                                                             │
│  ALL users must see these in order: Post → Comment → Reply  │
│                                                             │
│  But concurrent posts from User C can appear anywhere       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### How to Achieve It

**1. Vector Clocks / Version Vectors**

```python
class VectorClock:
    def __init__(self, num_nodes):
        self.clock = [0] * num_nodes
    
    def increment(self, node_id):
        self.clock[node_id] += 1
    
    def merge(self, other):
        for i in range(len(self.clock)):
            self.clock[i] = max(self.clock[i], other.clock[i])
    
    def happens_before(self, other):
        # Returns True if self causally precedes other
        return (all(s <= o for s, o in zip(self.clock, other.clock)) and
                any(s < o for s, o in zip(self.clock, other.clock)))
```

**2. Dependency Tracking**

```
Write Operation:
1. Include vector clock in write
2. Track which writes this write depends on
3. Replicate with dependency metadata

Read Operation:
1. Check if all dependencies are satisfied locally
2. If not, wait or fetch missing dependencies
3. Return value only when causally consistent
```

**3. COPS Protocol (Clusters of Order-Preserving Servers)**

* Track dependencies explicitly
* Delay reads until dependencies are satisfied

#### Real-World Examples

* **MongoDB**: Causal consistency sessions
* **Cassandra**: Lightweight transactions with causal ordering
* **COPS**: Academic system for geo-replicated stores
* **Social Media Feeds**: Comments must appear after posts

#### Trade-offs

| Pros                               | Cons                              |
| ---------------------------------- | --------------------------------- |
| Preserves intuitive ordering       | Metadata overhead (vector clocks) |
| Better availability than strong    | Complex to implement correctly    |
| No coordination for concurrent ops | May need to wait for dependencies |

#### When to Use

✅ **Use causal consistency when:**

* **Social media feeds**: Comments must appear after the post they're replying to; likes after the content exists.
* **Collaborative applications**: Google Docs, Figma—edits that depend on previous edits must be ordered.
* **Chat / messaging systems**: Reply must appear after the message being replied to.
* **Multi-step workflows**: Approval must come after submission; shipping after payment.
* **Geo-distributed systems**: When you need better availability than linearizability but more guarantees than eventual.

❌ **Avoid when:**

* Strict global ordering is required (use sequential/linearizable)
* The overhead of tracking dependencies is too high for your latency budget
* Operations are truly independent with no causal relationships
* Simple key-value workloads where eventual consistency suffices

***

### 4. Read-Your-Writes (Read-Your-Own-Writes)

#### What It Is

A process will always see its own writes. After a write completes, subsequent reads by the **same process** will reflect that write.

```
┌─────────────────────────────────────────────────────────────┐
│                  READ-YOUR-WRITES                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  User updates profile photo                                 │
│       │                                                     │
│       ▼                                                     │
│  User refreshes page                                        │
│       │                                                     │
│       ▼                                                     │
│  User MUST see their new photo (not old cached version)     │
│                                                             │
│  Other users might temporarily see old photo (that's OK)    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### How to Achieve It

**1. Sticky Sessions**

```
┌──────────┐     ┌──────────────┐
│  Client  │────►│ Load Balancer│
└──────────┘     └──────┬───────┘
                        │ Route to same replica
                        ▼
              ┌─────────────────┐
              │   Replica A     │ (client's writes and reads)
              └─────────────────┘
```

**2. Version Tracking**

```python
class ReadYourWritesClient:
    def __init__(self):
        self.last_write_timestamp = 0
    
    def write(self, key, value):
        response = self.send_write(key, value)
        self.last_write_timestamp = response.timestamp
        return response
    
    def read(self, key):
        # Include minimum version requirement
        return self.send_read(key, min_version=self.last_write_timestamp)
```

**3. Write to Leader, Read from Leader**

```
All reads/writes for a user go through the primary replica
```

#### Real-World Examples

* **Web Sessions**: PHP sessions, sticky cookies
* **DynamoDB**: Strongly consistent reads option
* **User Profile Updates**: Social media platforms
* **Shopping Cart**: E-commerce applications

#### Trade-offs

| Pros                         | Cons                               |
| ---------------------------- | ---------------------------------- |
| Intuitive user experience    | Limits load balancing              |
| Relatively easy to implement | Can create hot spots               |
| Good for user-facing apps    | Doesn't help with cross-user reads |

#### When to Use

✅ **Use read-your-writes when:**

* **User profile updates**: User changes their name/photo and must see it immediately on refresh.
* **Shopping cart**: Add item → view cart must show the item.
* **Form submissions**: Submit data → redirect to confirmation page must show submitted data.
* **Content creation**: Post a tweet/photo → must appear in your own feed immediately.
* **Settings changes**: Toggle a preference → UI must reflect the change right away.
* **Any user-facing write followed by read**: The "refresh and see my change" expectation.

❌ **Avoid when:**

* Cross-user consistency matters more (e.g., two users must see same inventory)
* Backend batch processing where user isn't waiting for feedback
* Analytics/reporting where slight delay is acceptable

***

### 5. Monotonic Reads

#### What It Is

Once a process reads a value, it will **never see an older value** in subsequent reads. Time doesn't go backward for that process.

```
┌─────────────────────────────────────────────────────────────┐
│                    MONOTONIC READS                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Valid Sequence:                                            │
│    Read(x) = 5 → Read(x) = 5 → Read(x) = 7 → Read(x) = 7    │
│              ✓              ✓              ✓                │
│                                                             │
│  Invalid Sequence:                                          │
│    Read(x) = 5 → Read(x) = 7 → Read(x) = 5                  │
│              ✓              ✗ (went backward!)              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### How to Achieve It

**1. Track Read Version**

```python
class MonotonicReadsClient:
    def __init__(self):
        self.read_versions = {}  # key -> last_seen_version
    
    def read(self, key):
        min_version = self.read_versions.get(key, 0)
        response = self.send_read(key, min_version=min_version)
        self.read_versions[key] = max(min_version, response.version)
        return response.value
```

**2. Sticky Sessions to Progressively Updated Replica**

```
Route reads to replicas that are at least as up-to-date as previously read
```

**3. Quorum Reads with Version Check**

```python
def monotonic_read(key, last_seen_version):
    # Read from multiple replicas
    responses = [replica.read(key) for replica in quorum]
    # Filter to those >= last_seen_version
    valid = [r for r in responses if r.version >= last_seen_version]
    # Return latest among valid
    return max(valid, key=lambda r: r.version)
```

#### Real-World Examples

* **News Feeds**: Don't show older posts after showing newer ones
* **Stock Tickers**: Price should not jump backward
* **Progress Indicators**: Download progress shouldn't decrease

#### Trade-offs

| Pros                             | Cons                             |
| -------------------------------- | -------------------------------- |
| Prevents confusing UX            | Requires version tracking        |
| Easy to understand               | May need to reject some replicas |
| Composable with other guarantees | Slight read latency increase     |

#### When to Use

✅ **Use monotonic reads when:**

* **Infinite scroll / pagination**: User scrolls down, then up—should never see "older" content than before.
* **Progress indicators**: Download 50% → 60% → never show 55% again.
* **Stock prices / live data**: Price tickers should never show older prices after showing newer ones.
* **Version displays**: "Last updated: 3pm" should never go backward to "2pm".
* **Leaderboards / rankings**: User's rank shouldn't fluctuate backward confusingly.
* **Read replicas**: When load balancing across replicas, ensure user doesn't hit a lagging replica after a fresh one.

❌ **Avoid when:**

* Data is naturally unordered (e.g., search results ranked by relevance)
* Showing historical snapshots intentionally ("data as of yesterday")
* The overhead of version tracking isn't worth it for non-critical data

***

### 6. Monotonic Writes

#### What It Is

Writes from a single process are applied in the order they were issued. A write is only applied after all previous writes from that process are applied.

```
┌─────────────────────────────────────────────────────────────┐
│                    MONOTONIC WRITES                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  User actions:                                              │
│    1. Create account                                        │
│    2. Set password                                          │
│    3. Add profile info                                      │
│                                                             │
│  Must be applied in order: 1 → 2 → 3                        │
│  Cannot apply password before account exists!               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### How to Achieve It

**1. Per-Client Sequence Numbers**

```python
class MonotonicWritesClient:
    def __init__(self, client_id):
        self.client_id = client_id
        self.sequence_num = 0
    
    def write(self, key, value):
        self.sequence_num += 1
        return self.send_write(
            key=key,
            value=value,
            client_id=self.client_id,
            seq=self.sequence_num
        )

class Server:
    def __init__(self):
        self.client_sequences = {}  # client_id -> last_applied_seq
        self.pending = {}  # client_id -> list of pending writes
    
    def apply_write(self, write):
        expected_seq = self.client_sequences.get(write.client_id, 0) + 1
        if write.seq == expected_seq:
            # Apply immediately
            self.do_apply(write)
            self.client_sequences[write.client_id] = write.seq
            # Check pending queue
            self.process_pending(write.client_id)
        elif write.seq > expected_seq:
            # Buffer for later
            self.pending.setdefault(write.client_id, []).append(write)
```

**2. Session-Based Ordering**

```
All writes in a session go to same partition/leader
Leader applies them in received order
```

#### Real-World Examples

* **Database Transactions**: Operations within transaction
* **Event Sourcing**: Events must be applied in order
* **Collaborative Editing**: User's edits in sequence

#### Trade-offs

| Pros                          | Cons                                   |
| ----------------------------- | -------------------------------------- |
| Preserves write intent        | Requires buffering out-of-order writes |
| Natural for many applications | Head-of-line blocking                  |
| Easy to reason about          | Session affinity needed                |

#### When to Use

✅ **Use monotonic writes when:**

* **User registration flow**: Create account → set password → add profile. Order matters!
* **Event sourcing**: Events must be applied in the order they were generated.
* **File operations**: Create directory → create file inside it. Can't reverse.
* **Database migrations**: Schema changes must apply in sequence.
* **Dependency chains**: Install package A before package B that depends on A.
* **Audit trails**: Actions must be logged in the order they occurred for compliance.

❌ **Avoid when:**

* Writes are independent and commutative (e.g., incrementing different counters)
* High-throughput scenarios where ordering overhead is too expensive
* Writes can be safely reordered without semantic impact

***

### 7. Bounded Staleness (Time-Bounded Consistency)

#### What It Is

Reads are guaranteed to return data that is **at most T seconds old** (or K versions behind). A middle ground between strong and eventual consistency.

```
┌─────────────────────────────────────────────────────────────┐
│                   BOUNDED STALENESS                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Configuration: max_staleness = 5 seconds                   │
│                                                             │
│  Timeline:                                                  │
│  ─────────────────────────────────────────────────────────  │
│  T=0    T=1    T=2    T=3    T=4    T=5    T=6              │
│   │      │      │      │      │      │      │               │
│   W(x=1) │      │      │      │      │      │               │
│          │      W(x=2) │      │      │      │               │
│          │      │      │      │      │  Read(x)             │
│          │      │      │      │      │    │                 │
│          │      │      │      │      │    ▼                 │
│                                       Must return x=2       │
│                                       (x=1 is too stale)    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### How to Achieve It

**1. Timestamp-Based Validation**

```python
class BoundedStalenessStore:
    def __init__(self, max_staleness_seconds):
        self.max_staleness = max_staleness_seconds
    
    def read(self, key):
        value, write_timestamp = self.local_replica.read(key)
        current_time = time.time()
        
        if current_time - write_timestamp > self.max_staleness:
            # Too stale, fetch from primary
            return self.fetch_from_primary(key)
        return value
```

**2. Vector Clock with Time Bounds**

```python
def is_acceptable(data_version, current_time, max_staleness):
    version_time = data_version.timestamp
    return (current_time - version_time) <= max_staleness
```

**3. Periodic Sync with Freshness Guarantee**

```
┌─────────────────┐         ┌─────────────────┐
│    Primary      │◄───────►│    Replica      │
│                 │  sync   │                 │
│                 │  every  │  max_lag = 5s   │
│                 │   1s    │                 │
└─────────────────┘         └─────────────────┘
```

#### Real-World Examples

* **Azure Cosmos DB**: Bounded staleness consistency level
* **Google Cloud Spanner**: Stale reads with timestamp bounds
* **CDN Cache**: TTL-based content freshness
* **Analytics Dashboards**: "Data as of 5 minutes ago"

#### Trade-offs

| Pros                             | Cons                                |
| -------------------------------- | ----------------------------------- |
| Configurable latency/consistency | Need synchronized clocks            |
| Good for read-heavy workloads    | Complex to implement correctly      |
| Predictable staleness            | May still return stale during bound |

#### When to Use

✅ **Use bounded staleness when:**

* **Analytics dashboards**: "Data refreshed every 5 minutes" is acceptable and expected.
* **Leaderboards**: Scores updated within X seconds is good enough.
* **Search indexes**: Documents indexed within N seconds of creation.
* **Recommendation systems**: Slightly stale user preferences are fine.
* **CDN caching**: Content can be stale for TTL duration.
* **Read replicas with SLA**: "Replica lag < 10 seconds" as a guarantee.
* **Regulatory requirements**: Some compliance requires data freshness within bounds.

❌ **Avoid when:**

* Staleness bound cannot be defined or tolerated
* Clock synchronization is unreliable across nodes
* Users expect real-time data (use stronger consistency)
* Write-heavy workloads where staleness bound is frequently violated

***

### 8. Eventual Consistency

#### What It Is

The **weakest** consistency model. If no new updates are made, eventually all replicas will converge to the same value. No guarantees about when or what intermediate states are seen.

```
┌─────────────────────────────────────────────────────────────┐
│                   EVENTUAL CONSISTENCY                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Write x=1 at Replica A                                     │
│       │                                                     │
│       │ (asynchronous replication)                          │
│       ▼                                                     │
│  Time passes...                                             │
│       │                                                     │
│       │  Different replicas may show:                       │
│       │  - Replica A: x=1                                   │
│       │  - Replica B: x=? (old value or empty)              │
│       │  - Replica C: x=? (old value or empty)              │
│       │                                                     │
│       ▼                                                     │
│  Eventually (undefined when):                               │
│    All replicas: x=1                                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### How to Achieve It

**1. Asynchronous Replication**

```python
class EventuallyConsistentStore:
    def write(self, key, value):
        # Write locally
        self.local_store[key] = value
        # Async replicate (fire and forget)
        self.replication_queue.put((key, value))
        return "OK"  # Return immediately
    
    def read(self, key):
        # Read from local replica (may be stale)
        return self.local_store.get(key)
```

**2. Conflict Resolution Strategies**

```
┌─────────────────────────────────────────────────────────────┐
│           CONFLICT RESOLUTION STRATEGIES                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Last-Writer-Wins (LWW)                                  │
│     - Highest timestamp wins                                │
│     - Simple but can lose data                              │
│                                                             │
│  2. Multi-Value (Siblings)                                  │
│     - Keep all conflicting values                           │
│     - Application resolves on read                          │
│                                                             │
│  3. CRDTs (Conflict-free Replicated Data Types)             │
│     - Mathematically guaranteed convergence                 │
│     - Counters, Sets, Registers                             │
│                                                             │
│  4. Custom Merge Functions                                  │
│     - Application-specific logic                            │
│     - Example: Union for sets, max for counters             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**3. CRDTs Example (G-Counter)**

```python
class GCounter:
    """Grow-only counter that eventually converges"""
    def __init__(self, node_id, num_nodes):
        self.node_id = node_id
        self.counts = [0] * num_nodes
    
    def increment(self):
        self.counts[self.node_id] += 1
    
    def value(self):
        return sum(self.counts)
    
    def merge(self, other):
        """Merge is commutative, associative, idempotent"""
        for i in range(len(self.counts)):
            self.counts[i] = max(self.counts[i], other.counts[i])
```

**4. Anti-Entropy Protocols**

```
┌────────────┐                    ┌────────────┐
│  Node A    │                    │  Node B    │
│            │                    │            │
│ data: {    │    Gossip/Sync     │ data: {    │
│   x: 1     │◄──────────────────►│   x: ?     │
│   y: 2     │                    │   y: 2     │
│ }          │                    │   z: 3     │
└────────────┘                    └────────────┘
              After sync:
              Both have {x:1, y:2, z:3}
```

#### Real-World Examples

* **Amazon DynamoDB**: Default consistency mode
* **Cassandra**: Tunable consistency (default: eventual)
* **DNS**: Propagation takes time
* **CDN**: Cached content across edge nodes
* **Social Media Likes**: Count may be temporarily inconsistent

#### Trade-offs

| Pros                 | Cons                           |
| -------------------- | ------------------------------ |
| Highest availability | May read stale data            |
| Lowest latency       | Conflict resolution needed     |
| Partition tolerant   | Hard to reason about           |
| Scales horizontally  | Not suitable for all use cases |

#### When to Use

✅ **Use eventual consistency when:**

* **Social media likes/views**: Exact count doesn't matter; approximate is fine.
* **DNS**: Propagation delay is inherent and acceptable.
* **Shopping cart (non-checkout)**: Browsing cart can be stale; only checkout needs consistency.
* **User activity tracking**: Pageviews, clicks—losing a few is acceptable.
* **Content delivery**: Cached static assets across CDN nodes.
* **Session data**: Non-critical session attributes.
* **Metrics / telemetry**: Aggregated data where precision isn't critical.
* **Global scale systems**: When availability and latency trump consistency.

❌ **Avoid when:**

* **Financial transactions**: Can't have inconsistent account balances.
* **Inventory management**: Overselling due to stale reads is costly.
* **Unique constraints**: Duplicate usernames, double bookings.
* **Coordination tasks**: Leader election, distributed locks.
* **User expectations demand freshness**: "I just posted this, where is it?"

**Pro tip**: Many systems use eventual consistency as the default and selectively strengthen consistency for specific operations (e.g., DynamoDB's strongly consistent reads, Cassandra's tunable consistency per query).

***

### Consistency Comparison Matrix

| Model                 | Coordination        | Latency     | Availability         | Use Case                    |
| --------------------- | ------------------- | ----------- | -------------------- | --------------------------- |
| **Linearizable**      | Consensus           | High        | Low during partition | Leader election, locks      |
| **Sequential**        | Total order         | Medium-High | Medium               | Event logs, queues          |
| **Causal**            | Dependency tracking | Medium      | High                 | Social feeds, collaboration |
| **Read-Your-Writes**  | Session tracking    | Low-Medium  | High                 | User-facing apps            |
| **Monotonic Reads**   | Version tracking    | Low         | High                 | Progress tracking           |
| **Monotonic Writes**  | Sequence numbers    | Low         | High                 | Ordered operations          |
| **Bounded Staleness** | Time sync           | Low         | High                 | Analytics, caching          |
| **Eventual**          | None                | Lowest      | Highest              | High-scale, global          |

***

### Choosing the Right Consistency Level

```
┌────────────────────────────────────────────────────────────────────────────────┐
│                   DECISION FLOWCHART                                           │
├────────────────────────────────────────────────────────────────────────────────┤
│                                                                                │
│  Need absolute ordering?                                                       │
│       │                                                                        │
│       ├── Yes ──► Need real-time? ──► Yes ──► Linearizable                     │
│       │                    │                                                   │
│       │                    └── No ──► Sequential                               │
│       │                                                                        │
│       └── No ──► Need causal ordering?                                         │
│                        │                                                       │
│                        ├── Yes ──► Causal Consistency                          │
│                        │                                                       │
│                        └── No ──► User-facing?                                 │
│                                       │                                        │
│                                       ├── Yes ──►                              │
│                                       │   Read-Your-Writes                     │
│                                       │   + Monotonic Reads                    │
│                                       │                                        │
│                                       └── No ──►                               │
│                                           Acceptable stale?                    │
│                                               │                                │
│                                               ├── Bounded ──► Bounded Staleness│
│                                               │                                │
│                                               └── Any ──► Eventual consistency │
│                                                                                │
└────────────────────────────────────────────────────────────────────────────────┘
```

***

### Session Guarantees Composition

Multiple session guarantees can be combined:

```python
class SessionGuarantees:
    """Combine multiple consistency guarantees per session"""
    
    def __init__(self):
        self.last_write_version = 0    # Read-your-writes
        self.last_read_version = {}    # Monotonic reads
        self.write_sequence = 0        # Monotonic writes
        self.causal_dependencies = []  # Causal consistency
    
    def write(self, key, value):
        self.write_sequence += 1
        version = self.send_write(
            key, value,
            sequence=self.write_sequence,
            dependencies=self.causal_dependencies
        )
        self.last_write_version = version
        self.causal_dependencies.append(version)
        return version
    
    def read(self, key):
        min_version = max(
            self.last_write_version,           # Read-your-writes
            self.last_read_version.get(key, 0) # Monotonic reads
        )
        result = self.send_read(key, min_version=min_version)
        self.last_read_version[key] = result.version
        self.causal_dependencies.append(result.version)
        return result.value
```

***

### Summary

| Consistency Level     | Key Guarantee          | When to Use                  |
| --------------------- | ---------------------- | ---------------------------- |
| **Linearizable**      | Appears as single copy | Distributed locks, consensus |
| **Sequential**        | Same order everywhere  | Message queues, logs         |
| **Causal**            | Respects causality     | Social apps, collaboration   |
| **Read-Your-Writes**  | See your own writes    | User sessions                |
| **Monotonic Reads**   | No going backward      | Progress, feeds              |
| **Monotonic Writes**  | Writes in order        | Dependent operations         |
| **Bounded Staleness** | Known max lag          | Analytics, caching           |
| **Eventual**          | Converges eventually   | High scale, global           |

**Remember**: There's no "best" consistency model—only the right trade-off for your specific requirements. Start with the weakest consistency that meets your needs and strengthen only where necessary.

***

### Further Reading

* **Designing Data-Intensive Applications** by Martin Kleppmann
* **Consistency Models** - Jepsen.io
* **CAP Theorem** - Brewer's original paper
* **CRDTs** - Shapiro et al.
* **Spanner: Google's Globally Distributed Database**
* **Dynamo: Amazon's Highly Available Key-value Store**
