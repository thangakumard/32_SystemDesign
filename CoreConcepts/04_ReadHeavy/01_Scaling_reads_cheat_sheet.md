# System Design Cheat Sheet: Scaling Reads

### Architectural Strategies and Frameworks for Handling High-Volume Read Workloads

---

## Table of Contents
* [1. The Scaling Reads Core Framework](#1-the-scaling-reads-core-framework)
* [2. Database Optimizations (First Line of Defense)](#2-database-optimizations-first-line-of-defense)
  * [Indexing Strategies](#indexing-strategies)
  * [Vertical Scaling](#vertical-scaling)
  * [Denormalization](#denormalization)
* [3. Horizontal Database Scaling](#3-horizontal-database-scaling)
  * [Read Replicas](#read-replicas)
  * [Database Sharding (Horizontal Partitioning)](#database-sharding-horizontal-partitioning)
    * [Sharding Strategies](#sharding-strategies)
* [4. External Caching Layers](#4-external-caching-layers)
  * [Application-Level Caching](#application-level-caching)
  * [CDN & Edge Caching](#cdn--edge-caching)
* [5. Common Interview Scenarios & Mitigation Strategies](#5-common-interview-scenarios--mitigation-strategies)

---

## 1. The Scaling Reads Core Framework

When read traffic vastly outpaces write traffic (common read-to-write ratios range from 10:1 to 100:1+ in platforms like Instagram, YouTube, or Twitter), single databases hit **physical limits** (CPU execution speeds, memory capacity, and disk I/O bottlenecks). 

To handle massive read volume, the system architecture must progress sequentially:
1. Maximize efficiency within the existing database engine.
2. Distribute data horizontally across multiple database nodes.
3. Introduce in-memory and edge caching layers to reduce database trips.

---

## 2. Database Optimizations (First Line of Defense)

Before adding complex distributed architecture, maximize the performance of the single-instance database.

### Indexing Strategies
* **Mechanism:** Speed up query lookups by maintaining B-Trees or Hash indexes on frequently filtered or sorted columns (e.g., `user_id`, `created_at`).
* **Pros:** Significant performance boost with zero architectural overhead.
* **Cons:** Slower write speeds, increased memory and storage requirements.

### Vertical Scaling
* **Mechanism:** Upgrade hardware components (CPU, RAM, NVMe SSDs) on the primary database machine.
* **Pros:** Simple, immediate capacity extension without code changes.
* **Cons:** Hard hardware limits, expensive at scale, single point of failure (SPOF).

### Denormalization
* **Mechanism:** Store duplicate data or pre-calculated aggregated metrics (such as comment counts or like tallies) directly on primary records to eliminate expensive real-time relational `JOIN` operations.
* **Pros:** Blazing-fast read queries.
* **Cons:** Complex write workflows to maintain consistency across tables.

---

## 3. Horizontal Database Scaling

When a single database instance cannot process the read throughput or hold the dataset size, distribute the database across multiple physical or virtual machines.

### Read Replicas
* **Mechanism:** Route write operations exclusively to a **Primary Database**, which asynchronously or synchronously streams transaction logs to multiple **Read Replicas**. Read requests are load-balanced across these replicas.
* **Best Used For:** High read-to-write ratio workloads.
* **Trade-offs:** **Replication Lag**. Asynchronous replication can delay updates across replicas, causing eventual consistency issues (e.g., a user posts a comment but cannot immediately see it upon refreshing).

### Database Sharding (Horizontal Partitioning)
* **Mechanism:** Break large, monolithic database tables into smaller, independent chunks called **shards**, and distribute them across independent host nodes.

#### Sharding Strategies
1. **Range-Based Partitioning:**
   * Partition data based on ranges of key values (e.g., User IDs 1–1,000,000 on Shard 1; 1,000,001–2,000,000 on Shard 2).
   * *Risk:* Prone to hot-spotting if traffic concentrates within specific ranges.
2. **Hash-Based Partitioning:**
   * Hash a shard key using a function like `hash(key) % total_shards` to scatter records evenly across host nodes.
   * *Risk:* Adding or removing shards requires expensive data rebalancing unless Consistent Hashing is applied.
3. **Directory-Based Partitioning:**
   * Use a centralized lookup service that maps keys to specific shard locations.
   * *Risk:* The directory lookup service can become a single point of failure or performance bottleneck.

* **Key Trade-offs of Sharding:** 
  * High operational complexity.
  * Cross-shard queries and `JOIN` operations become exceptionally slow or unsupported.
  * Re-sharding and schema migrations are difficult to execute online.

---

## 4. External Caching Layers

Keep hot read queries out of the database entirely by serving data from high-speed memory or geographical edge locations.

### Application-Level Caching
* **Tools:** Redis, Memcached.
* **Patterns:**
  * **Cache-Aside (Lazy Loading):** Application queries the cache first. On a miss, it fetches from the DB, populates the cache, and returns the response.
  * **Write-Through / Write-Back:** Write traffic updates or asynchronously syncs with the database via the cache layer.
* **Mitigations:** Use mutex locks or probabilistic early expiration to prevent **Cache Stampedes / Thundering Herds** when hot keys expire.

### CDN & Edge Caching
* **Tools:** Cloudflare, AWS CloudFront, Fastly.
* **Mechanism:** Offload static assets (images, media, static HTML snippets, API JSON responses) to edge servers close to the user.
* **Pros:** Minimizes latency and shields origin servers from direct traffic spikes.

---

## 5. Common Interview Scenarios & Mitigation Strategies

| Scenario | Architectural Solution |
| :--- | :--- |
| **Queries slow down as data size grows** | Introduce composite indexing, evaluate query execution plans, and implement horizontal database sharding. |
| **Users see stale data after updates** | Address replication lag by routing critical post-write reads (e.g., user profile updates) directly to the **Primary Database**. |
| **Cache miss causes database collapse** | Prevent cache stampedes using distributed locks (mutexes) or pre-warming hot key TTLs. |
| **Hot shard receiving unequal traffic** | Re-shard using consistent hashing or switch to a hash/salt combination on the shard key. |