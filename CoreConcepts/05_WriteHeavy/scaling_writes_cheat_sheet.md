# System Design Cheat Sheet: Scaling Writes
*A practical guide to architecting high-throughput, write-heavy distributed systems based on the Hello Interview framework.*

---

## Table of Contents
1. [Overview: The Core Challenge](#1-overview-the-core-challenge)
    * [Why Reads Are Easy vs. Why Writes Are Hard](#why-reads-are-easy-vs-why-writes-are-hard)
2. [The 4-Pillar Scaling Writes Framework](#2-the-4-pillar-scaling-writes-framework)
    * [Pillar 1: Vertical Scaling & Write Optimization](#pillar-1-vertical-scaling--write-optimization)
    * [Pillar 2: Sharding and Partitioning](#pillar-2-sharding-and-partitioning)
    * [Pillar 3: Handling Bursts with Queues & Load Shedding](#pillar-3-handling-bursts-with-queues--load-shedding)
    * [Pillar 4: Batching and Hierarchical Aggregation](#pillar-4-batching-and-hierarchical-aggregation)
3. [Common Interview Deep Dives](#3-common-interview-deep-dives)
    * [Handling Resharding and Scaling Out](#handling-resharding-and-scaling-out)
    * [Mitigating Hot Key Bottlenecks](#mitigating-hot-key-bottlenecks)

---

## 1. Overview: The Core Challenge

### Why Reads Are Easy vs. Why Writes Are Hard
* **Read Heavy Systems:** Scaling reads is relatively simple because reading data is **idempotent** and **stateless**. You can scale reads almost endlessly by adding read replicas, CDNs, or aggressive caching layers (like Redis or Memcached).
* **Write Heavy Systems:** Scaling writes is inherently **stateful**. Every write mutates data, requires strict consistency guarantees (e.g., ACID transactions), and frequently hits physical hardware limitations like **disk I/O, CPU contention, and network bandwidth**.

When designing write-heavy applications, the key goals are to maximize throughput, minimize latency, and handle traffic spikes without locking the database or losing data.

---

## 2. The 4-Pillar Scaling Writes Framework

To scale writes effectively beyond a single node, engineers rely on four core architectural strategies:

### Pillar 1: Vertical Scaling & Write Optimization
*Before introducing distributed complexity, optimize the single node.*

* **Vertical Scaling:** Upgrade hardware specs (more CPU cores, more RAM, faster NVMe SSDs to boost IOPS).
* **LSM-Tree Storage Engines:** Use databases backed by **Log-Structured Merge-trees** (e.g., Cassandra, RocksDB, ScyllaDB). Instead of performing random write operations to disk like B-Trees (standard relational databases), LSM engines append writes sequentially to an in-memory buffer (**MemTable**) and a **WAL (Write-Ahead Log)**, which avoids disk-head contention and dramatically accelerates write speeds.
* **NoSQL / Wide-Column Stores:** Trade off immediate ACID transactional guarantees or complex relational schemas for high-speed, scalable write operations.

### Pillar 2: Sharding and Partitioning
*Distribute the write load across multiple database nodes when a single instance hits physical limits.*

* **Horizontal Sharding:** Distribute data across separate, independent database nodes using a **Shard Key** (e.g., `user_id`, `tenant_id`). All write traffic for a given shard key routes directly to its assigned node, spreading parallel writes across the cluster.
* **Vertical Partitioning:** Split database tables by columns (e.g., separating frequently updated status fields or logs from static user profile information) to reduce individual row sizes and minimize locking contention.

### Pillar 3: Handling Bursts with Queues & Load Shedding
*Smooth out unpredictable traffic spikes (e.g., flash sales, breaking news events).*

* **Write Queues / Asynchronous Processing:** Insert a distributed message queue (e.g., Apache Kafka, RabbitMQ, AWS SQS) between the API layer and the database. The system acknowledges the request immediately to the user with a `202 Accepted` status, while background consumer workers pull from the queue at a sustainable rate for the database.
* **Load Shedding & Rate Limiting:** Implement rate limiters at the API gateway level to reject excess traffic gracefully (returning HTTP `429 Too Many Requests`) when write queues reach maximum capacity, protecting system stability.

### Pillar 4: Batching and Hierarchical Aggregation
*Reduce network chatter and database round-trips by consolidating operations.*

* **Batching:** Combine multiple write operations into a single network payload or bulk `INSERT` query (e.g., batching log entries or analytics events every 1,000 writes or 500ms).
* **Hierarchical Aggregation:** Use an in-memory layer (like Redis) or a stream processing engine (like Apache Flink) to aggregate high-frequency metrics in memory before writing a single consolidated sum to persistent storage (e.g., accumulating ad clicks or video views in memory every second before issuing a single update to PostgreSQL).

---

## 3. Common Interview Deep Dives

### Handling Resharding and Scaling Out
**Question:** *"How do you handle resharding when your current dataset outgrows existing shards?"*

* **Consistent Hashing:** Route keys using a hash ring to minimize the remapping of keys across nodes when expanding the cluster size.
* **Live Data Migration / Dual Writing:** 
  1. Spin up new shard nodes.
  2. Start writing new updates simultaneously to both old and new shards (**Dual Writing** or **Change Data Capture (CDC)**).
  3. Backfill historical data from the old shards to the new shards in the background.
  4. Cut over read/write routing to the new shards once fully synchronized.

### Mitigating Hot Key Bottlenecks
**Question:** *"What happens when a single 'hot key' receives millions of concurrent writes?" (e.g., a viral social post or high-profile account)*

* **Salted / Partitioned Keys:** Append a random salt or bucket integer to the primary key (e.g., transform `post_123` into `post_123_bucket_1`, `post_123_bucket_2`, ..., `post_123_bucket_10`). Writes are uniformly distributed across distinct shard partitions, and reads aggregate across all buckets on demand.
* **In-Memory Write Buffering:** Intercept writes for the hot key in an in-memory cache or message stream, aggregate the counts or updates locally, and perform periodic background writes to the primary database.