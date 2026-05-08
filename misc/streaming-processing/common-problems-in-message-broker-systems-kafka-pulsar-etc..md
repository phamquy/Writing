---
icon: face-tissue
---

# Common Problems in Message Broker Systems (Kafka, Pulsar, etc.)

Message brokers like **Apache Kafka** and **Apache Pulsar** are powerful for building distributed, event-driven systems.\
However, they often encounter recurring challenges due to their scale, complexity, and distributed nature.

## 0. Refresher

When a broker accepts a message it appends the payload to local storage, transforming the topic into a strictly ordered log. Consumer groups (or Pulsar subscriptions) traverse that log independently and at their own pace.

Because these logs are conceptually infinite, brokers enforce retention windows: data stays on disk only until the configured horizon unless tiered storage moves aged segments elsewhere.

Each consumer group typically fans out across several workers, with every worker owning one or more partitions. Once a worker finishes processing a record it commits the offset, telling the broker to advance the pointer and expose the next event.

## 1. Backlog increase & Backpressure

* **Problem:** Consumers fall behind producers, creating large backlogs.
* **Impact:** Delayed processing, increased memory/disk usage, potential SLA violations.
* **Cause:** Slow consumers, uneven partition distribution, or insufficient resources.

This probably the most common issue in operating system with a message broker. To troubleshoot this we need to understand a life time of a message.

Backlog issue can be caused by any point betwee both broker and consumer side.

### Broker host is under pressure

**On the broker side**, overloaded hosts (CPU, RAM, or thread pool exhaustion) simply cannot hand messages to consumers fast enough. When that happens the only durable fix is usually to add broker capacity. Just remember that every time you add or remove brokers the cluster must rebalance partitions, which temporarily pauses consumption and can trigger redeliveries.

To reduce the blast radius, scale out in small increments:

**Rebalance Impact**:

* Shorter rebalance duration with fewer partition reassignments
* Less processing downtime per scaling event
* Easier to monitor and rollback if issues occur

**Stability**:

* Gradual load distribution changes
* Time to observe performance impact between scales
* Reduced risk of overwhelming brokers or consumers

**Operational Benefits**:

* Easier troubleshooting if problems arise
* Better capacity planning and monitoring
* Lower blast radius if scaling causes issues

**Example Approach**: Instead of: 5 consumers → 20 consumers (4x scale) Do: 5 → 8 → 12 → 16 → 20 consumers (incremental)

**Considerations**:

* **Partition limits** - can't have more consumers than partitions (kafka specific)
* **Rebalance frequency** - allow time between scales (30-60 seconds minimum)
* **Monitoring** - watch lag, throughput, and error rates between scales

**Exception**: Large scale-up might be acceptable during planned maintenance windows when brief downtime is acceptable, but incremental scaling is still safer for production traffic.

Disk I/O can also be the culprit, even if it shows up less often. Pulsar brokers rely on Apache BookKeeper for durable storage, so every publish and fetch adds an extra hop that can become a bottleneck. When you see saturated disks or spiking read/write ops, revisit broker and BookKeeper I/O settings—batch sizes, cache, tiered storage—to smooth the path.

### Network congestion

Network congestion can be a root cause of Kafka and Pulsar backlog buildup.

* **How Network Congestion Causes Backlogs**
  * **Producer slowdown** - High network latency or packet loss forces producers to _retry_ sends, reducing throughput
  * **Consumer lag** - Consumers can't fetch messages fast enough due to network bottlenecks, causing messages to accumulate
  * **Replication delays** - _Inter-broker replication slows down_, affecting partition leadership and availability
  * **Coordinator communication issues** - Consumer group coordination and offset commits become unreliable
* **Common Network-Related Scenarios**
  * Cross-AZ or cross-region deployments with insufficient bandwidth
  * Shared network infrastructure with competing traffic
  * Network interface saturation on broker nodes
  * DNS resolution delays affecting service discovery
  * Load balancer bottlenecks in front of brokers
* **Diagnostic Indicators**
  * High _network utilization_ metrics on broker hosts
  * Increased _request timeouts_ in producer/consumer logs
  * Growing _replication lag_ between brokers
  * Elevated _connection establishment times_
  * _Packet loss or retransmission_ metrics
* **Mitigation Strategies**
  * Monitor network bandwidth and latency between components
  * Use dedicated network paths for messaging traffic
  * Implement proper network QoS policies
  * Scale network capacity or optimize topology
  * Tune client timeout and retry configurations
  * Consider message batching to reduce network overhead

_**Caution**_: aggressive batching reduces network overhead but can hammer disk I/O—especially in Pulsar’s storage layer. Always load-test different batch sizes to find a safe midpoint.

### Slow consumer

This probably the most common cause of the backlog build up. Consumers are basically your application, there can be million things can go wrong in general, there are thing in your control and not. Let assume your code is bug free. Then what can slow down consumers?:

#### Underscaled consumers

Most backlog situations simply indicate an underscaled consumer fleet—the traffic volume spiked and the workers cannot drain partitions quickly enough. You will usually see this in host-level metrics first (CPU, memory, JVM heap). When those gauges peg, increase consumer capacity before the lag snowballs.

Horizontal scaling tactics depend on the broker and subscription style:

* **Kafka:** Each partition is owned by exactly one consumer in a group, so you cannot out-scale the partition count. Plan for future growth by over-partitioning topics up front so you have headroom.
* **Pulsar:** `Shared` subscriptions let you add consumers freely because message order does not matter, while `KeyShared` can scale but forces key redistribution. `Exclusive` and `Failover` mirror Kafka’s one-partition-per-consumer limit.

Vertical scaling (bigger instances, faster disks) is still an option and can be cheaper than horizontal fan-out when workloads stay steady but need more per-node headroom.

#### Downstream regressing

Consumers often depend on downstream systems (databases, APIs, caches) to complete processing. If those services slow down or become unavailable, consumers back up waiting for responses.

Downstream regressions are inevitable in distributed systems and usually fall outside your direct control. You probably already have global playbooks for retries, exponential backoff, fallbacks, or read-only mode, so we will not rehash those. What matters for stream processing is translating those guardrails into consumer behavior—throttling fetches, pausing partitions, or offloading work to dead-letter queues before lag swamps the cluster.

And if possible (if we own the producer as well) then we should consider slow down producer as well. To avoid the high pressure at a single point in the pipeline (producer -> broker -> consumer), each component in the pipeline should respect the backpressure signal from the downstream, and slow down, pause or buffer (to a storage?).

In broker-mediated communication, backpressure signals are sent through different mechanisms since there's no direct connection between producer and consumer.

**Broker-Level Backpressure Mechanisms**

* Producer-Side Signals
  * **Broker queue full** - Broker rejects or blocks new messages
  * **Flow control** - Broker throttles producer send rate
  * **Memory/disk pressure** - Broker returns resource exhaustion errors
  * **Network congestion** - TCP backpressure from broker to producer
* Consumer-Side Signals
  * **Slow acknowledgments** - Consumer delays message acks to signal overload
  * **Reduced fetch rates** - Consumer decreases polling frequency
  * **Consumer lag metrics** - Exposed via monitoring for upstream awareness

**Implementation Patterns**

* Circuit Breaker Pattern
  * Producer monitors:
    * Broker response times
    * Error rates
    * Queue depth metrics → Reduces send rate or stops producing

**Rate Limiting**

* Producer implements:
  * Token bucket algorithms
  * Sliding window rate limits
  * Dynamic rate adjustment based on broker feedback
* Consumer Feedback Loops
  * Consumer publishes metrics: Processing latency, Queue depth, Error rates → Upstream services monitor and adjust

**Broker-Specific Mechanisms**

* Kafka
  * **Producer buffer full** - Blocks or drops messages
  * **Broker quotas** - Throttles based on client ID
  * **Consumer lag monitoring** - JMX metrics for upstream awareness
* Pulsar
  * **Flow control** - Built-in producer rate limiting
  * **Backlog quotas** - Automatic producer throttling
  * **Consumer acknowledgment delays** - Natural backpressure signal
* RabbitMQ
  * **Memory alarms** - Blocks all publishing connections
  * **Flow control** - Per-connection credit-based system
  * **Queue length limits** - Rejects messages when full

**Monitoring-Based Backpressure** Metrics-Driven Approach

* **Queue depth monitoring** - Upstream adjusts based on broker metrics
* **Consumer lag alerts** - Trigger upstream rate limiting
* **End-to-end latency** - System-wide backpressure coordination

The key difference from direct calls is that backpressure becomes asynchronous and metrics-driven rather than immediate and connection-based.

***

## 2. Message Loss or Duplication

* **Problem:** Messages may be lost or delivered more than once.
* **Impact:** Data inconsistency, broken workflows.
* **Cause:** Misconfigured acknowledgment settings, broker crashes, or improper producer retries.

Messages flow from producer to broker, sit on broker storage until consumers fetch them, and eventually age out per the retention policy (or get offloaded to tiered storage). Once a consumer processes and acknowledges a record, the broker typically omits it from future deliveries.

### Message loss

Every stage of that journey introduces its own loss modes. Below we examine each hop assuming healthy application code—only external failures (power, hardware, network, region-wide disasters) are in scope.

#### Producer <-> Broker

**Crash 💥**

If the producer or broker dies mid-flight (power loss, node crash, network partition) durability depends entirely on the acknowledgment policy. In Kafka, setting `acks=0` means the producer never waits for confirmation, while `acks=1` only waits for the leader. In both cases a leader crash before the write hits disk drops the record. Pulsar behaves similarly: `sendAsync()` returns immediately, so if the broker fails before persisting the entry, that message evaporates.

**Mitigations**

* Tighten ack requirements (`acks=all`, Pulsar synchronous send) whenever you cannot tolerate that risk. If `acks=all` slow thing down, then at least we need to config min number of replica ack greater than `1`
* Producer should have retry strategy (with backoff)

**Network failures ⛓️‍💥**

If a message is in flight when the network flakes out, the producer may never see the broker’s acknowledgment. The write might already be durable—or it might have vanished—but without strict ack policies and retries the producer assumes success and the record is lost. From the producer’s vantage point, transient network partitions look exactly like broker crashes, so apply the same mitigations: stronger acks, and bounded retries with backoff.

One caveat: if the producer retries after the broker actually persisted the write, you now have duplicates. Guard against that with idempotent producers and deduplication logic downstream so processing remains idempotent.

**Write failure ✍️**

Brokers can reject writes for many reasons: disk full, quota exceeded, authorization failure, malformed messages, etc. If the producer does not handle those errors properly (retrying when safe, failing fast when not), messages can slip through the cracks.

Kafka leans on the OS page cache, so data is at risk until it is flushed to disk. Pulsar delegates durability to BookKeeper ledgers, yet any quorum failure still drops writes. Brokers refuse to ack when persistence fails, but it is on the producer to notice the error, respect its ack policy, and retry safely.

#### Broker <-> Consumer

In Kafka, most consumer-side loss stems from careless offset management. Auto-commit happily advances offsets before your code finishes processing, so any crash after the commit but before persistence skips work forever. Even with manual commits, crashes between fetch and commit can still mark messages as done if the logic is sloppy. Slow consumers run afoul of retention windows—the broker expires partitions and the lagging client never sees the missing data. Rebalances add another wrinkle: when partitions move between group members, stale offsets or missed commits translate into gaps. Finally, deserialization failures often get swallowed unless the client surfaces them, giving the illusion that the record never existed. The cure is boring but effective: disable auto-commit, checkpoint offsets only after durable processing, monitor lag versus retention, and treat deserialization errors as first-class failures routed to dead-letter queues.

Pulsar pushes loss prevention onto explicit acknowledgments and subscription semantics. In `Exclusive` and `Failover` modes only one consumer sees a partition, so a crash without an ack leaves the message invisible until the broker redelivers or the client restarts. `Shared` subscriptions spread load but make it easy to misconfigure acknowledgment timeouts or DLQs, causing silent drops when messages exceed redelivery counts. Non-persistent topics exacerbate the issue because messages never touch disk—disconnect for a moment and they evaporate. Tight receiver queues under heavy throughput can also overflow and discard entries before the application touches them. Without a properly wired DLQ, those failures disappear into logs. Stick to persistent topics when durability matters, size `receiverQueueSize` to match throughput, configure DLQs with sane redelivery limits, and ensure the client acknowledges only after the downstream write succeeds.

Across both systems, the mitigation checklist is similar: observe consumer lag and retention headroom, surface errors rather than swallowing them, and ensure retries or DLQs preserve at-least-once guarantees. The exact APIs differ, but the principle is universal—never declare a message complete until your business logic truly finishes, and always provide a deterministic escape hatch for poison records.

* Ensure consumers handle errors gracefully.
* Monitor lag, retries, and redelivery metrics.

***

## 3. Scaling Complexity

* **Problem:** Adding brokers or partitions can cause rebalancing storms.
* **Impact:** Temporary downtime, uneven load distribution, degraded performance.
* **Cause:** Kafka’s partition rebalancing or Pulsar’s topic ownership migration.

Partition math is destiny in both ecosystems. Each Kafka partition is pinned to exactly one broker replica set, so every time you add capacity you must reshuffle those partitions. If you expand too aggressively, thousands of leader elections and replica movements saturate the controller and your clients see timeouts. Pulsar hides partitions behind bundles, but splitting and migrating those bundles still pauses traffic: the topic owner hands off to another broker, BookKeeper replicas rebalance, and consumers see a blip. Scaling BookKeeper itself has the same problem—new storage nodes force ledger re-replication before they become useful.

Rebalancing storms usually start with good intentions: “let’s double broker count before the holiday rush.” The cluster then spends hours copying terabytes of data while producers throttle and consumers stall. Worse, if the new brokers are mis-sized or misconfigured, you might have to roll back and endure the same pain again. Plan for incremental expansions instead. Kafka’s `kafka-reassign-partitions.sh` supports phased plans; Pulsar’s namespace bundle splitting can be staged bundle-by-bundle so you only disturb a subset of topics at a time.

Multi-region or multi-tenant growth adds another dimension. Kafka users often shard by business domain, spinning up new clusters per region because cross-region replication (MirrorMaker 2, Cluster Linking) is itself heavy. Pulsar encourages tenant-level isolation through namespaces, but you still need to pre-provision sufficient bundles and BookKeeper ensemble sizes per tenant. Under-sizing either forces emergency repartitioning later, which is the least convenient moment to discover your automation playbooks are outdated.

**Mitigation playbook**

* Model partition-to-broker placement before you scale; simulate disk and network impact of replica moves.
* Automate staged expansion (drain → add broker → move limited partitions → validate) instead of one massive rebalance.
* Align BookKeeper ensemble/replication settings with broker growth so storage does not become the hidden bottleneck.
* Keep consumers informed: advertise maintenance windows, monitor lag, and pause non-critical workloads while the cluster reshapes.

***

## 4. Metadata Dependencies

* **Problem:** Reliance on external metadata services (ZooKeeper, BookKeeper).
* **Impact:** Cluster instability if metadata services fail.
* **Cause:** Misconfigured quorum, network partitions, or slow metadata updates.

Kafka’s modern releases demoted ZooKeeper but did not remove the metadata problem: the new KRaft controllers still require a healthy quorum to elect leaders and publish partition states. Any hiccup—split-brain, slow disk, GC pause—blocks metadata updates, and suddenly brokers refuse to accept produce/fetch requests because they cannot confirm leadership. Legacy ZooKeeper deployments add even more moving parts: observers, chroot paths, JAAS configs, ACLs. Human error (wrong `myid`, skewed clocks, missing TLS certs) routinely takes down the control plane.

Pulsar doubles down on metadata services. ZooKeeper tracks namespaces, tenants, and broker ownership; BookKeeper stores ledgers and fencing info; brokers cache both. Lose quorum in either layer and writes halt. Even when services stay up, slow disks on BookKeeper nodes inflate write latency and make brokers think a ledger is unhealthy, triggering redundant failovers. Because metadata updates are frequent (cursor positions, topic ownership, ledger creation), any network partition or poor TLS setup between these services surfaces immediately as publish/consume failures.

Hardening the metadata tier pays the highest dividends:

* Treat controller/ZooKeeper/BookKeeper clusters as first-class: dedicated hardware, identical configs, aggressive monitoring of quorum health, disk latency, and heap usage.
* Automate certificate rotation and client ACL provisioning so human error does not brick the control plane.
* Load-test metadata at scale—simulate thousands of topic creations and cursor moves—to ensure default timeouts are realistic.
* Keep the clusters geographically close to brokers; high RTT between metadata and brokers is a silent throughput killer.

***

## 5. Operational Overhead

* **Problem:** Complex cluster management (brokers, storage nodes, metadata services).
* **Impact:** Higher maintenance cost, risk of misconfiguration.
* **Cause:** Multi-component architecture (Kafka: brokers + ZooKeeper/KRaft; Pulsar: brokers + BookKeeper + ZooKeeper).

Operating a serious messaging stack means running a small distributed systems zoo. Kafka may look simpler on paper, but in practice you juggle brokers, controller nodes, schema registries, MirrorMaker fleets, Connect workers, and the CI/CD that upgrades all of them. Pulsar raises the stakes with its broker + BookKeeper + ZooKeeper split, plus optional functions workers, proxies, and tiered-storage offloaders. Each component has its own JVM options, disk layout, TLS keystores, and capacity planning model. Missing a single config flag (e.g., BookKeeper journal on the wrong disk, Kafka log cleaner disabled) can undo months of reliability work.

Day-2 operations are where teams burn out: patching CVEs, rotating certs, resizing disks, replaying failed tiered-storage uploads, and proving compliance. Troubleshooting becomes a game of “which component is lying?” because metrics are scattered across Prometheus jobs, broker logs, BookKeeper logs, and client applications. Without ruthless automation, your ops team becomes the serialization point for every internal customer who wants a new topic or retention tweak.

To keep overhead sane:

* Standardize cluster templates (Terraform/Ansible/Kubernetes operators) so you can recreate environments deterministically.
* Centralize configuration management and drift detection; never hand-edit broker configs in production.
* Provide self-service tooling for product teams (topic catalogs, quota dashboards) so ops is not the bottleneck.
* Budget for observability: unified tracing from producer → broker → storage, log aggregation, and runbooks that explain what each alert means.
* When possible, push low-value workloads to managed offerings (Confluent Cloud, StreamNative) so the team focuses on the parts that must stay in-house.

***

## 6. Latency Spikes

* **Problem:** Sudden increases in message delivery time.
* **Impact:** Breaks real-time guarantees.
* **Cause:** Disk I/O bottlenecks, GC pauses, or network congestion.

Latency jitters usually show up first as angry dashboards and “message stuck?” tickets. Brokers are write-ahead logs, so any component that slows disk flushes or network replication elongates end-to-end latency. In Kafka, a single broker with a slow disk controller can stall an entire partition because the leader must fsync before responding. ISR shrink/expand events then cascade to followers and to the controller. Pulsar inherits all of those risks plus BookKeeper, whose ledger writes must land on multiple bookies; if one bookie GC pauses or experiences high IO wait, the append call blocks even though the broker is healthy.

Another common culprit is garbage collection. JVM brokers tuned for throughput often run large heaps with G1 or Shenandoah. Surprise: a mis-sized region or off-heap buffer leak still creates stop-the-world hiccups. You see it as periodic latency spikes aligned with GC logs. TLS everywhere amplifies the issue because expensive handshakes and certificate validations happen on the hot path whenever connections churn.

Client behavior matters too. Producers that batch aggressively can amplify tail latency under light load (waiting to fill batch linger), while consumers that fetch huge batches may block application threads processing the next chunk. Network appliances (load balancers, firewalls doing deep packet inspection) inject micro-bursts of delay whenever they rotate flows.

**How to tame the spikes**

* Benchmark disks and isolate broker logs, snapshots, and BookKeeper journals on separate devices; enable async disk IO where supported.
* Keep JVM GC logging on at INFO, tie alerts to pause time budgets, and prefer newer collectors (ZGC, Shenandoah) for large heaps.
* Use adaptive batching: cap linger.ms / batch size on Kafka producers, enable Pulsar’s `batchingMaxPublishDelay` tuning, and monitor p99 publish latency.
* Run network QoS or DSCP tagging for broker traffic so “chatty” services cannot starve replication links; avoid hairpinning through unnecessary load balancers.
* Measure end-to-end latency (producer timestamp to consumer processing) and break it down per hop so you know whether the broker, storage, or client logic is guilty.

***

## 7. Security & Multi-Tenancy

* **Problem:** Misconfigured ACLs or tenant isolation.
* **Impact:** Unauthorized access, data leakage.
* **Cause:** Weak authentication/authorization setup, especially in multi-tenant Pulsar clusters.

Security on paper is a handful of TLS certs and ACL files; in production it is a never-ending game of “who touched my topic.” Kafka exposes listeners per protocol (PLAINTEXT, SASL, TLS). Forget to disable PLAINTEXT internally and suddenly every pod on the VPC can publish. Misapplied ACL patterns (“User:\* has Write on Topic:\*”) are another classic foot gun. Pulsar adds tenants and namespaces, which is great for isolation until someone reuses the same token across environments or forgets to scope a role to the right namespace, effectively giving one tenant the keys to another’s backlog.

Multi-tenancy also magnifies blast radius. A noisy tenant that bypasses quotas can saturate broker CPU, BookKeeper I/O, or ZooKeeper connections, starving well-behaved tenants. Cross-tenant data leaks are even nastier: misconfigured proxies or shared auth secrets may let a tenant subscribe to someone else’s topic. Because Pulsar supports per-tenant auth, proxies, and limits, the configuration surface is large—easy to drift.

On the Kafka side, migrations to SASL/OAUTHBEARER or mTLS often stall because certificate rotation is manual and the Connect ecosystem lags behind broker features. Pulsar’s token- or OAuth-based auth requires synchronizing issuers between brokers, proxies, and functions workers; a mismatch leads to mysterious 401 errors that operators misdiagnose as broker bugs.

**Practical safeguards**

* Enforce TLS everywhere and automate cert/key rotation; use short-lived certs issued by an internal CA to limit blast radius.
* Treat ACLs/roles as code: store in Git, review via pull requests, and deploy through automation rather than kubectl exec.
* Define per-tenant quotas (throughput, storage, concurrent connections) so one customer cannot exhaust the shared control plane.
* Instrument audit logging for produce/consume events and management API calls; ship them to a tamper-resistant store.
* In Pulsar, isolate sensitive tenants with dedicated namespaces, proxies, and BookKeeper ensembles when compliance demands hard walls.
* Regularly pen-test broker endpoints and rotate service credentials; stale SASL passwords are a common pivot point for attackers.

***

## 8. Takeaways

If there is a single pattern across all these failure modes, it is that brokers faithfully amplify whatever discipline—or lack of it—you impose upstream. Plan capacity in advance, over-partition so you can scale gradually, and watch lag like a hawk. Lock down acknowledgments end to end, treat retries and DLQs as first-class citizens, and never mark work complete before downstream writes stick. Harden the control plane (ZooKeeper, BookKeeper, controllers) as if it were your revenue service, because it is. Finally, automate everything: scaling runs, cert rotations, ACL changes, topic provisioning, and latency dashboards. The brokers will still surprise you, but at least the surprises will be observable, isolated, and reversible.
