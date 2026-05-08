---
icon: up-right-and-down-left-from-center
---

# Scaling up Kafka and Pulsar consumers

## Pulsar Consumer Scaling Without Repartitioning

Pulsar consumers can scale up without repartitioning, but it depends on the subscription type and current partition count.

### Subscription Types and Scaling

#### Shared Subscription

* **Can scale beyond partitions** - Multiple consumers can consume from the same partition
* **No repartitioning needed** - Pulsar handles message distribution automatically
* **Dynamic scaling** - Add/remove consumers at runtime without disruption

#### Key\_Shared Subscription

* **Can scale beyond partitions** - Messages with same key go to same consumer
* **Maintains ordering per key** - No repartitioning required
* **Automatic rebalancing** - Pulsar redistributes key ranges when consumers change

#### Exclusive/Failover Subscription

* **Limited by partition count** - One consumer per partition maximum
* **Requires repartitioning** - To scale beyond current partition count
* **Ordering guarantees** - Maintains strict message ordering per partition

### Scaling Scenarios

#### Without Repartitioning

Topic: 4 partitions \
Shared subscription: Can have 10+ consumers \
Key\_Shared subscription: Can have 10+ consumers (with key distribution)

#### Requires Repartitioning

Topic: 4 partitions \
Exclusive subscription: Maximum 4 consumers \
Failover subscription: Maximum 4 consumers

### Best Practices

* Use Shared or Key\_Shared subscriptions for horizontal scaling
* Monitor consumer lag and throughput metrics
* Consider partition count during topic creation for future exclusive/failover needs
* Leverage Pulsar's built-in load balancing for shared subscriptions

Pulsar's flexible subscription model makes it easier to scale consumers compared to traditional partitioned systems like Kafka.

## Kafka Consumer Scaling

Kafka consumers cannot scale beyond the number of partitions without repartitioning.

### Kafka's Partition-Consumer Relationship

#### Hard Limitation

* **One consumer per partition maximum** - Within a consumer group
* **Cannot exceed partition count** - If you have 4 partitions, maximum 4 active consumers
* **Idle consumers** - Extra consumers in the group remain idle

#### Scaling Scenarios

Topic: 4 partitions \
Consumer Group: Can have maximum 4 active consumers \
Additional consumers: Will be idle until rebalancing occurs

### Workarounds (Without Repartitioning)

#### Multiple Consumer Groups

* **Different consumer groups** - Each can have up to partition-count consumers
* **Same messages consumed multiple times** - Each group processes all messages
* **Use case** - Different applications processing same data

#### Consumer Threading

* **Single consumer, multiple threads** - Process messages in parallel within consumer
* **Manual thread management** - Handle threading and message processing yourself
* **Ordering considerations** - May lose per-partition ordering guarantees

### When Repartitioning is Required

* **Horizontal scaling beyond partitions** - Need more parallel consumers
* **Increased throughput demands** - Current partition count insufficient
* **Performance bottlenecks** - Individual partitions becoming hotspots

### Best Practices

* **Plan partition count upfront** - Consider future scaling needs
* **Monitor consumer lag** - Identify when scaling is needed
* **Use appropriate partition key** - Ensure even distribution
* **Consider operational overhead** - Repartitioning requires careful planning

Unlike Pulsar's flexible subscription models, Kafka's design tightly couples consumers to partitions, making scaling more restrictive.
