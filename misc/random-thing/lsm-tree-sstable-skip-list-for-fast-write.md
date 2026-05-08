---
icon: database
---

# LSM Tree, SSTable, Skip List for fast write

## 🌲 Log-Structured Merge Tree (LSM Tree)

An **LSM Tree** is a disk-based data structure optimized for **fast writes**. It's commonly used in modern databases like **LevelDB**, **RocksDB**, **Cassandra**, and **HBase**.

***

### 🔧 How LSM Tree Works

#### 1. **MemTable (In-Memory)**

* New writes go to:
  * A **Write-Ahead Log (WAL)** for durability.
  * A **MemTable**, an in-memory **sorted** structure (e.g., Skip List).
* Once full, the MemTable is **flushed** to disk as an **SSTable** (Sorted String Table).

#### 2. **SSTables (On Disk)**

* SSTables are **immutable, sorted** files on disk.
* New data doesn't overwrite—new SSTables are created.

#### 3. **Compaction**

* Periodically, SSTables are **merged** to:
  * Eliminate duplicates.
  * Apply deletions (tombstones).
  * Reduce the number of SSTables.
* Improves read performance and space efficiency.

#### 4. **Levels**

* SSTables are organized in **levels**:
  * **Level 0**: Most recent data, may overlap in key ranges.
  * **Level 1+**: Larger files, less overlap, more space-efficient.

***

### 🔍 Read Path

1. **Search MemTable**.
2. **Search SSTables**, starting from most recent (Level 0).
3. Use **Bloom Filters** to skip unnecessary files.
4. If needed, search disk files in order.

***

### ✅ Advantages

* High **write throughput**.
* Good for **sequential** I/O patterns.
* Easy to compress data during compaction.

### ❌ Disadvantages

* **Read amplification**: Many SSTables might need to be checked.
* **Write amplification**: Data rewritten multiple times during compaction.
* **Latency spikes** during background compaction.

***

### 📦 Popular Systems Using LSM Trees

* LevelDB / RocksDB
* Apache Cassandra
* HBase
* ClickHouse

