# Schema Evolution in Distributed Systems

In the lifecycle of any long-lived distributed system, change is the only constant. As business requirements shift, the structure of the data—the **schema**—must evolve. Schema evolution is the process of managing these changes to ensure that data remains accessible and meaningful across different versions of services and over time.

### 1. The Problem with Schema Evolution

The core problem is that data producers (writers) and data consumers (readers) change at different rates. In a monolithic application, you might migrate the database and update the application code simultaneously during a downtime window. In a distributed system, this is rarely possible.

* **Versioning Mismatch:** A service writing data might be on version `v2`, while a service reading it is still on `v1`.
* **Data Corruption:** If a field is removed or its type changed without coordination, deserialization failures or silent data corruption can occur.
* **Downtime:** Poorly managed schema changes often require "stop-the-world" maintenance, which violates high-availability SLAs.

### 2. Challenges in Distributed Systems

Distributed environments introduce specific complexities to schema management:

#### Decoupled Deployments (Rolling Updates)

Services are deployed independently. During a rolling update, you will have a mix of old and new nodes running simultaneously.

* **Old Consumer, New Producer:** Can the old code handle data with new fields?
* **New Consumer, Old Producer:** Can the new code handle data missing new fields?

#### Data in Motion vs. Data at Rest

* **Data in Motion (RPC, Queues):** Ephemeral. The schema compatibility matters primarily during the transition period of deployment.
* **Data at Rest (Databases, Data Lakes):** Persistent. You might read a record written 5 years ago. The current schema must still be able to interpret that old binary blob (Backward Compatibility is critical).

#### Fan-out Architectures

A single event in a message broker (like Kafka) might be consumed by ten different downstream services. You cannot upgrade all ten consumers simultaneously. The schema must remain compatible with the "lowest common denominator" consumer.

### 3. Best Practices in Schema Management

To navigate these challenges, engineers rely on strict compatibility guarantees and management patterns.

#### Compatibility Modes

* **Backward Compatibility:** New code can read data written by old code. (Essential for reading historical data).
* **Forward Compatibility:** Old code can read data written by new code. (Essential for rolling upgrades where producers upgrade before consumers).
* **Full (Transitive) Compatibility:** Both backward and forward compatible. This is the gold standard for distributed systems.

#### The "Golden Rules" of Evolution

1. **Add Optional Fields Only:** When adding a field, give it a default value. Old readers will ignore it; new readers will see the default if reading old data.
2. **Never Rename Fields:** Renaming is technically a "Delete" followed by an "Add". It breaks compatibility unless handled in multiple phases.
3. **Never Change Types:** Changing a `string` to an `int` is usually a breaking change.
4. **Avoid `Required` Fields:** In systems like Protobuf, `required` fields are dangerous because removing them later is a breaking change.

### 4. Concrete Examples by Schema Type

#### A. Database Schema (SQL)

* **Scenario:** Adding a `phone_number` column to a `Users` table.
* **Bad Practice:** `ALTER TABLE Users ADD COLUMN phone_number VARCHAR(20) NOT NULL;`
  * _Why:_ This locks the table and fails if existing rows exist. It also breaks running code that inserts into the table without knowing about the new column.
* **Best Practice:**
  1. Add column as `NULLable`.
  2. Update code to write to the new column.
  3. Backfill data for old rows.
  4. (Optional) Add `NOT NULL` constraint later.
* **Tools:** Flyway, Liquibase.

#### B. RPC Schema (Protobuf, Thrift)

* **Scenario:** Adding an `email` field to a `UserRequest` message.
*   **Protobuf Example:**

    ```protobuf
    // Version 1
    message UserRequest {
      string name = 1;
      int32 id = 2;
    }

    // Version 2 (Evolution)
    message UserRequest {
      string name = 1;
      int32 id = 2;
      string email = 3; // New field with new tag ID
    }
    ```
* **Mechanism:** Protobuf uses integer tags (`= 1`, `= 2`) to identify fields on the wire, not names.
  * _Forward Compatibility:_ An old reader sees tag `3`, doesn't recognize it, and ignores it (stores it in "unknown fields").
  * _Backward Compatibility:_ A new reader reads old data (missing tag `3`) and assigns the default value (empty string).

#### C. Message Broker Schema (Avro with Kafka)

Avro is popular in the Hadoop/Kafka ecosystem because the schema is usually stored separately (in a Registry) rather than sent with every message (to save space).

* **Scenario:** A producer sends `LogEvent` to a Kafka topic.
* **Avro Evolution:**
  * **Writer Schema:** The schema used to serialize the data.
  * **Reader Schema:** The schema used by the consumer to deserialize.
  * **Resolution:** Avro resolves differences. If the Reader expects a field `ip_address` with a default value, but the Writer didn't send it, Avro fills it in.
* **Enforcement:** The Schema Registry rejects a producer trying to register a schema that is incompatible with previous versions.

### 5. Available Tools and Systems

#### Schema Registries

Centralized services that store versioned schemas and enforce compatibility rules.

* **Confluent Schema Registry:** The standard for Kafka. Supports Avro, Protobuf, and JSON Schema. It prevents "bad" schemas from entering the pipeline.
* **AWS Glue Schema Registry:** Serverless registry for AWS managed services (Kinesis, MSK).
* **Pulsar Schema Registry:** Built directly into Apache Pulsar.

#### Database Migration Tools

* **Flyway / Liquibase:** Java-based tools that manage SQL migration scripts versioning.
* **Atlas (HashiCorp):** Modern tool for managing DB schemas as code.
* **Gh-ost / pt-online-schema-change:** Tools for MySQL to perform schema changes without locking tables (online DDL).

#### Serialization Libraries

* **Protocol Buffers (Google):** Highly efficient, strong typing, excellent backward/forward compatibility support via tags.
* **Apache Avro:** JSON-defined schema, binary payload. Best for dynamic typing and big data storage.
* **Apache Thrift:** Similar to Protobuf, used heavily at Meta/Uber.
* **Cap'n Proto / FlatBuffers:** Zero-copy serialization, also support schema evolution.
