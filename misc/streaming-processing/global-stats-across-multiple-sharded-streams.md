---
icon: chart-bullet
---

# Global stats across multiple sharded streams

One of the common operations on data streams is to calculate some stat on the stream’s data (normally over a time window). Performing this operation in one consumer (single process) is simple as the consumer has a global view of the data stream. However, things get trickier when you scale; you normally need to partition or shard your stream into multiple partitions, and you will need multiple consumers to process your stream. Each consumer now no longer has a global view of your stream data, so how can you calculate a stat over all data from the stream? \
\
Let’s take an example where we need to calculate the top-k value from a stream. This document gives a quick explanation of how to compute a global Top-K in AWS Kinesis Data Analytics (or a similar approach on other streaming platforms) when your application has multiple parallel consumer subtasks.

### Background

When your KDA application has Parallelism > 1 (e.g., 2 subtasks):

* Each subtask consumes a subset of shards from the source stream.
* Each subtask processes events independently.
* Aggregations like COUNT or SUM are local to each subtask unless you merge results later.

To compute a Top-K globally, you need a two-stage aggregation or a global operator that combines partial results from all subtasks.

***

### Option 1: Two-Stage SQL Aggregation (KDA SQL)

```
-- Step 1: Partial aggregation (parallel)
CREATE OR REPLACE STREAM partial_counts (
  item VARCHAR(64),
  cnt BIGINT
);

CREATE OR REPLACE PUMP partial_pump AS
INSERT INTO partial_counts
SELECT
  item,
  COUNT(*) AS cnt
FROM source_stream
GROUP BY item;

-- Step 2: Global aggregation (single task)
CREATE OR REPLACE STREAM global_topk AS
SELECT STREAM item, SUM(cnt) AS total_cnt
FROM partial_counts
GROUP BY item
ORDER BY total_cnt DESC
LIMIT 10;
```

Key Points:

* partial\_counts collects local counts per subtask.
* The second stage merges counts across all subtasks.
* No partition key in the second stage → KDA routes all rows to a single task.

***

### Option 2: Apache Flink DataStream API (KDA Flink)

```
DataStream<Event> events = ...; // from Kinesis source

// Local pre-aggregation
DataStream<Tuple2<String, Long>> localCounts = events
    .keyBy(Event::getItem)
    .window(TumblingProcessingTimeWindows.of(Time.minutes(1)))
    .aggregate(new CountAggregator());

// Global merge (single parallel instance)
DataStream<Tuple2<String, Long>> globalTopK = localCounts
    .windowAll(TumblingProcessingTimeWindows.of(Time.minutes(1)))
    .process(new TopNFunction(10)); // custom Top-N logic
```

Notes:

* .windowAll(...) creates a global window.
* TopNFunction computes the Top-K globally.
* For high-scale streams, you can merge local Top-K sets in a second stage instead of a full global aggregation.

***

### Option 3: External Merge for Large Scale

* Subtasks write partial aggregates to S3 or DynamoDB.
* A secondary process (AWS Glue, Lambda, or another Flink job) computes the final Top-K.
* Useful when memory constraints prevent in-memory global aggregation.

***

### Summary Table

| Approach                     | Tool      | Global Correctness | Performance          | Typical Use                 |
| ---------------------------- | --------- | ------------------ | -------------------- | --------------------------- |
| Two-stage SQL aggregation    | KDA SQL   | ✅ Yes              | ⚡ Fast, simple       | Analytics queries           |
| windowAll global operator    | KDA Flink | ✅ Yes              | ⚠ Single parallelism | Stateful stream apps        |
| Partial Top-K merge          | KDA Flink | ✅ Yes              | ✅ Scalable           | High-volume leaderboards    |
| External merge (Glue/Lambda) | Any       | ✅ Yes              | 🐢 Batch-like        | Large scale / offline top-K |

The above approached ensures that Top-K computation accounts for all shards and all parallel subtasks in your Kinesis stream.
