---
icon: arrow-progress
---

# How Apache Flink distribute base on keyed windows?

## Apache Flink Distributed Processing Architecture: Understanding Hash-Based Partitioning and Network Mesh

This document explains how Apache Flink distributes work among cluster nodes using hash-based partitioning and network mesh architecture.

We will use example where source stream of tweets partitioned on language, and we need to count tweets base on hashtags.

### Overview

Apache Flink uses a sophisticated distribution strategy that involves two distinct layers:

1. **Source Level Distribution** - How data is ingested from external streams
2. **Keyed Stream Distribution** - How data is partitioned and processed based on keys

### Stream Partitioning and Key Distribution

When you create a keyed stream with hashtags as keys, Flink uses a **hash-based partitioning strategy**:

```
Tweet Stream → Hash(hashtag) → Partition Assignment
```

#### Key Principles:

* Each hashtag gets hashed using a consistent hash function (typically MurmurHash)
* The hash value determines which task slot (and thus which node) processes that hashtag
* All tweets with the same hashtag always go to the same task instance
* This ensures consistency for windowed aggregations

### Task Parallelism and Slot Distribution

Flink distributes tasks across the cluster using a slot-based architecture:

```
Flink Cluster Architecture:
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│   Node 1    │  │   Node 2    │  │   Node 3    │
│ ┌─────────┐ │  │ ┌─────────┐ │  │ ┌─────────┐ │
│ │ Slot 1  │ │  │ │ Slot 3  │ │  │ │ Slot 5  │ │
│ │ Slot 2  │ │  │ │ Slot 4  │ │  │ │ Slot 6  │ │
│ └─────────┘ │  │ └─────────┘ │  │ └─────────┘ │
└─────────────┘  └─────────────┘  └─────────────┘
```

#### Task Assignment Example:

* `#AI` → always goes to Slot 2 (Node 1)
* `#MachineLearning` → always goes to Slot 4 (Node 2)
* `#DataScience` → always goes to Slot 6 (Node 3)

Each slot maintains its own window state for assigned hashtags, and windows are created/triggered independently per key.

### Source vs Keyed Operators

Flink employs **two different distribution strategies** for different layers of processing:

#### Source Level (Tweet Stream Ingestion)

```
Tweet Stream → Source Operators (distributed across nodes)
```

**Characteristics:**

* Multiple nodes can subscribe to the source stream (similar to Kafka partition assignment)
* Each node reads from specific partitions of your tweet stream
* Distribution is based on source partitioning (not key-based)

#### Keyed Stream Level (Hashtag Processing)

```
Source Operators → Hash(hashtag) → Keyed Operators (specific nodes)
```

**Characteristics:**

* Upstream operators calculate the hash and forward to downstream operators
* Only the designated node processes each specific hashtag
* Distribution is based on consistent hashing of the key

### Hash Calculation and Forwarding

The hash calculation happens **at the upstream operator** (not at a central coordinator):

```typescript
/**
 * Demonstrates how hash calculation works in Flink's distribution
 * Each upstream task calculates where to send each hashtag
 */
export interface HashtagDistribution {
  /**
   * Calculates target task for a hashtag using consistent hashing
   * 
   * @param hashtag - the hashtag to be processed
   * @param parallelism - number of downstream tasks
   * @returns target task index
   * @throws Never throws - pure function
   */
  calculateTargetTask(hashtag: string, parallelism: number): number {
    // Flink uses MurmurHash or similar
    const hash = this.murmurHash(hashtag)
    return Math.abs(hash) % parallelism
  }
}
```

#### Distribution Process:

1. **Source operators** read from their assigned partitions
2. **Each hashtag** gets hashed by the source operator that read the containing tweet
3. **Hash result** determines which downstream task should process the hashtag
4. **Network transfer** occurs if the target task is on a different node

### Network Mesh Architecture

Flink creates a **network mesh** where any source node can forward data to any keyed operator node:

```
Tweet Stream Partitions:
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│ Partition 0 │  │ Partition 1 │  │ Partition 2 │
└─────┬───────┘  └─────┬───────┘  └─────┬───────┘
      │                │                │
      ▼                ▼                ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│   Node A    │  │   Node B    │  │   Node C    │
│ (Source Op) │  │ (Source Op) │  │ (Source Op) │
└─────┬───────┘  └─────┬───────┘  └─────┬───────┘
      │                │                │
      └────────────────┼────────────────┘
                       │
        Network Shuffle (All-to-All)
                       │
      ┌────────────────┼────────────────┐
      │                │                │
      ▼                ▼                ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│   Node A    │  │   Node B    │  │   Node C    │
│ (Keyed Op)  │  │ (Keyed Op)  │  │ (Keyed Op)  │
│   #AI       │  │   #Tech     │  │   #ML       │
└─────────────┘  └─────────────┘  └─────────────┘
```

#### Network Mesh Benefits:

1. **All-to-All Connectivity**: Every source node can send data to every keyed operator node
2. **Automatic Routing**: Flink handles the network routing based on hash calculations
3. **Consistent Hashing**: Same hashtag always goes to same downstream task, regardless of source
4. **Load Balancing**: Hash function distributes hashtags evenly across downstream tasks
5. **Fault Tolerance**: If a node fails, Flink can redistribute both source reading and keyed processing

### Practical Implementation Example

Here's how the distribution logic works in practice (simple code for illustration only):

```typescript
/**
 * Source operator on each node - reads assigned partitions
 * and forwards hashtags to appropriate downstream tasks
 * 
 * @throws Never throws - all errors converted to Result
 */
export class FlinkSourceOperator {
  private readonly nodeId: string
  private readonly assignedPartitions: number[]
  private readonly downstreamParallelism: number
  
  constructor(nodeId: string, assignedPartitions: number[], downstreamParallelism: number) {
    this.nodeId = nodeId
    this.assignedPartitions = assignedPartitions
    this.downstreamParallelism = downstreamParallelism
  }
  
  /**
   * Processes tweets from assigned partitions and calculates
   * which downstream task should handle each hashtag
   * 
   * @param tweets - tweets from assigned partitions
   * @returns Result containing hashtag distribution information
   * @throws Never throws - all errors converted to Result
   */
  async processTweets(tweets: Tweet[]): Promise<Result<HashtagDistribution[], NarvarErr>> {
    try {
      const distributions: HashtagDistribution[] = []
      
      for (const tweet of tweets) {
        for (const hashtag of tweet.hashtags) {
          const targetTaskIndex = this.calculateTargetTask(hashtag)
          distributions.push({
            hashtag,
            sourceNodeId: this.nodeId,
            targetTaskIndex,
            tweet
          })
        }
      }
      
      return Result.ok(distributions)
    } catch (error) {
      return Result.narvarErr(`Failed to process tweets: ${error}`)
    }
  }
  
  /**
   * Calculates which downstream task should handle this hashtag
   * Uses same hash function across all nodes for consistency
   * 
   * @param hashtag - hashtag to hash
   * @returns target task index
   * @throws Never throws - pure function
   */
  private calculateTargetTask(hashtag: string): number {
    const hash = this.murmurHash(hashtag)
    return Math.abs(hash) % this.downstreamParallelism
  }
  
  /**
   * Simplified MurmurHash implementation for demonstration
   * 
   * @param key - string to hash
   * @returns hash value
   * @throws Never throws - pure function
   */
  private murmurHash(key: string): number {
    let hash = 0
    for (let i = 0; i < key.length; i++) {
      const char = key.charCodeAt(i)
      hash = ((hash << 5) - hash) + char
      hash = hash & hash
    }
    return hash
  }
}

export type Tweet = {
  id: string
  text: string
  hashtags: string[]
  language: string
}

export type HashtagDistribution = {
  hashtag: string
  sourceNodeId: string
  targetTaskIndex: number
  tweet: Tweet
}
```

#### How Nodes Know Their Hash Assignment

Flink uses a **deterministic hash function** that all nodes know about:

```typescript
/**
 * All nodes use the same hash function and know the cluster topology
 * This ensures consistent routing decisions across the cluster
 */
export class FlinkHashAssignment {
  private readonly parallelism: number
  
  constructor(parallelism: number) {
    this.parallelism = parallelism
  }
  
  /**
   * Every node can calculate which task should handle any key
   * 
   * @param key - key to hash and assign
   * @returns target task index
   * @throws Never throws - pure function
   */
  getTargetTaskIndex(key: string): number {
    return Math.abs(this.hash(key)) % this.parallelism
  }
  
  /**
   * Each task knows its own index and can determine if it should process a key
   * 
   * @param key - key to check
   * @param myTaskIndex - current task's index
   * @returns true if this task should process the key
   * @throws Never throws - pure function
   */
  shouldProcessKey(key: string, myTaskIndex: number): boolean {
    return this.getTargetTaskIndex(key) === myTaskIndex
  }
  
  /**
   * Hash function implementation
   * 
   * @param value - value to hash
   * @returns hash result
   * @throws Never throws - pure function
   */
  private hash(value: string): number {
    let hash = 0
    for (let i = 0; i < value.length; i++) {
      const char = value.charCodeAt(i)
      hash = ((hash << 5) - hash) + char
      hash = hash & hash
    }
    return hash
  }
}
```

### Benefits and Challenges

#### Benefits:

* **Consistency**: Same hashtags always processed by same task
* **State Locality**: Window state is kept local to each task
* **Scalability**: Adding more nodes increases parallelism
* **Load Balancing**: Hash function distributes keys evenly
* **Automatic Management**: No manual partition assignment needed

#### Challenges:

1. **Hot Keys**: Popular hashtags (like `#COVID` or `#Election`) might create hotspots
2. **Skewed Distribution**: Some nodes might get more popular hashtags
3. **Memory Usage**: Nodes with popular hashtags need more state storage
4. **Network Overhead**: Cross-node communication for hash-based routing

#### Optimization Strategies:

* **Custom Partitioner**: Implement custom logic for better distribution
* **Salt Keys**: Add random prefixes to popular hashtags to spread load
* **Resource Allocation**: Assign more resources to nodes handling popular keys
* **Monitoring**: Track key distribution and resource usage

### Comparison with Traditional Approaches

#### Traditional Partition-Based Processing

```typescript
/**
 * In traditional setup (like current stream consumer approach):
 * - Node A reads from partition 0 only
 * - Node A processes only hashtags that happen to be in partition 0
 * - Hashtag distribution depends on original partitioning strategy
 */
export class TraditionalProcessing {
  /**
   * Each node processes whatever hashtags happen to be in its assigned partition
   * No guarantee that same hashtag always goes to same node
   * 
   * @throws May throw depending on partition assignment
   */
  processPartition(): void {
    // Node A: processes whatever hashtags are in partition 0
    // Node B: processes whatever hashtags are in partition 1  
    // Node C: processes whatever hashtags are in partition 2
    // Result: Uneven distribution, can't guarantee hashtag consistency
  }
}
```

#### Flink's Mesh-Based Processing

```typescript
/**
 * In Flink's mesh setup:
 * - Node A reads from partition 0, calculates hash, forwards to appropriate node
 * - Node B reads from partition 1, calculates hash, forwards to appropriate node
 * - Node C reads from partition 2, calculates hash, forwards to appropriate node
 * - ALL nodes can receive hashtags from ANY source node
 */
export class FlinkMeshProcessing {
  /**
   * Perfect hashtag consistency through hash-based routing
   * Even load distribution through consistent hashing
   * 
   * @param hashtag - hashtag to be processed
   * @param currentNodeId - node that read the original tweet
   * @param totalNodes - total nodes in keyed layer
   * @returns target node information
   * @throws Never throws - pure function
   */
  calculateMeshForwarding(hashtag: string, currentNodeId: string, totalNodes: number): MeshForwardingInfo {
    const targetNodeIndex = Math.abs(this.hash(hashtag)) % totalNodes
    const shouldForward = targetNodeIndex !== this.getNodeIndex(currentNodeId)
    
    return {
      hashtag,
      sourceNode: currentNodeId,
      targetNodeIndex,
      shouldForward,
      networkHop: shouldForward ? 'required' : 'local'
    }
  }
  
  /**
   * Hash function for consistent partitioning
   * 
   * @param value - value to hash
   * @returns hash result
   * @throws Never throws - pure function
   */
  private hash(value: string): number {
    let hash = 0
    for (let i = 0; i < value.length; i++) {
      const char = value.charCodeAt(i)
      hash = ((hash << 5) - hash) + char
      hash = hash & hash
    }
    return hash
  }
  
  /**
   * Converts node ID to numeric index
   * 
   * @param nodeId - node identifier
   * @returns numeric index
   * @throws Never throws - string parsing is safe
   */
  private getNodeIndex(nodeId: string): number {
    return parseInt(nodeId.replace('node-', ''))
  }
}

export type MeshForwardingInfo = {
  hashtag: string
  sourceNode: string
  targetNodeIndex: number
  shouldForward: boolean
  networkHop: 'required' | 'local'
}
```

### Key Architectural Points

1. **No Central Coordinator**: Each upstream task calculates hashes independently
2. **Deterministic**: Same hashtag always goes to same downstream task
3. **Network Shuffling**: Data moves between nodes based on hash calculations
4. **Automatic**: Flink JobManager coordinates the topology, but hash calculation is distributed
5. **Scalable**: Adding nodes increases both source reading and keyed processing capacity

### Conclusion

Flink's distributed architecture ensures that hashtag counting is both scalable and consistent through:

* **Hash-based partitioning** for consistent key assignment
* **Network mesh architecture** for flexible data routing
* **Automatic load balancing** through consistent hashing
* **State locality** for efficient windowed operations

This architecture guarantees that each hashtag's complete history is maintained on a single task for accurate windowed aggregations, while distributing the computational load evenly across the cluster.

The mesh network approach allows Flink to achieve better load distribution and consistency guarantees compared to traditional partition-based processing approaches, making it ideal for applications requiring both scalability and correctness in distributed stream processing scenarios.



{% hint style="info" %}
Now we see how Flink distribute work among node with consistent hash, but one important assumption that we have not explored here is that for that to work all the node in the cluster MUST have the same version of cluster topology (\~ who is the members of the cluster). See [apache-flink-architecture.md](streaming-processing/apache-flink-architecture.md "mention") to see how Flink ensure that
{% endhint %}



