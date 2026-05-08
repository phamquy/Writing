---
icon: comments-question-check
---

# Why Kinesis Data Analytics Has Limitations with Nested Arrays and Multiple Arrays

Amazon Kinesis Data Analytics (KDA) has restrictions on nested arrays and multiple arrays in the same record due to how its SQL-based streaming engine processes data.

## Kinesis Data Analytics Basics

* KDA allows SQL-like queries on streaming data.
* It ingests JSON or structured records.
* The SQL engine maps JSON fields into relational columns.
* Each field is treated as a scalar or a single array, not arbitrary nested structures.



## Why Nested Arrays Are Not Supported

Example of a nested array:

```
{
  "user": "alice",
  "activities": [
    {
      "type": "click",
      "timestamps": [123, 456]
    }
  ]
}
```

Challenges:

* Flattening difficulty: SQL operates on rows/columns, not hierarchical structures. Flattening activities.timestamps is ambiguous.
* Stream transformations: KDA expects each record to map cleanly to rows; nested arrays expand unpredictably.
* Performance & memory: Arbitrary nesting requires runtime expansion of multiple levels, which is expensive for high-throughput streams.<br>

## Why Two Arrays in the Same Record Are Not Supported

Example:

```
{
  "user": "alice",
  "clicks": [1,2,3],
  "views": [4,5,6]
}
```

Challenges:

* KDA can flatten only one array per record at a time.
* Multiple arrays require cross-product expansion:
  * clicks has 3 elements
  * views has 3 elements
  * Flattening both produces 3 × 3 = 9 rows
* This leads to data explosion and performance issues, which the engine avoids by design.

## Workarounds



* Preprocess the stream: Use Lambda, Kinesis Data Firehose, or Flink to flatten nested arrays before sending to KDA.
* Split records: Send each array as a separate record, so KDA deals with one array at a time.
* Use KDA for Apache Flink: Flink supports nested structures and multiple arrays naturally, offering a full programming model instead of only SQL.



## Summary Table

| Limitation                    | Reason                                                                                           |
| ----------------------------- | ------------------------------------------------------------------------------------------------ |
| Nested arrays                 | SQL engine cannot flatten multiple hierarchical levels easily; ambiguous mapping to rows/columns |
| Two arrays in the same record | Would require cross-product flattening, leading to data explosion and performance issues         |
