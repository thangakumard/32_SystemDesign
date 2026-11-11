# PostgreSQL Performance & Architecture: Comprehensive Cheat Sheet

*A complete reference guide for scaling PostgreSQL read and write performance, index optimization, consistency trade-offs, and architectural decision-making, based on the Hello Interview curriculum.*

## Table of Contents

* [1. Core Bottlenecks of Single-Node Postgres](#1-core-bottlenecks-of-single-node-postgres)
  * [1.1 CPU and Query Processing Limits](#11-cpu-and-query-processing-limits)
  * [1.2 Disk I/O and Buffer Pool Cache Misses](#12-disk-io-and-buffer-pool-cache-misses)
  * [1.3 Connection Limits and Overhead](#13-connection-limits-and-overhead)
  * [1.4 Lock Contention](#14-lock-contention)
* [2. Read Performance](#2-read-performance)
  * [2.1 Memory Architecture & Buffer Pool](#21-memory-architecture--buffer-pool)
  * [2.2 Connection Pooling (PgBouncer)](#22-connection-pooling-pgbouncer)
  * [2.3 Caching Layers & Strategies](#23-caching-layers--strategies)
  * [2.4 Denormalization & Materialized Views](#24-denormalization--materialized-views)
* [3. Basic Indexing](#3-basic-indexing)
  * [3.1 B-Tree Indexes (The Default)](#31-b-tree-indexes-the-default)
  * [3.2 Hash Indexes](#32-hash-indexes)
  * [3.3 Multi-Column (Composite) Indexes](#33-multi-column-composite-indexes)
* [4. Beyond Basic Indexes](#4-beyond-basic-indexes)
  * [4.1 Partial Indexes](#41-partial-indexes)
  * [4.2 Expression / Functional Indexes](#42-expression--functional-indexes)
  * [4.3 Covering Indexes (`INCLUDE` Clause)](#43-covering-indexes-include-clause)
  * [4.4 Specialized Index Types (GIN, GiST, BRIN)](#44-specialized-index-types-gin-gist-brin)
* [5. Query Optimization Essentials](#5-query-optimization-essentials)
  * [5.1 `EXPLAIN` and `EXPLAIN ANALYZE`](#51-explain-and-explain-analyze)
  * [5.2 Understanding Join Strategies](#52-understanding-join-strategies)
  * [5.3 Avoiding Common Anti-Patterns](#53-avoiding-common-anti-patterns)
  * [5.4 Statistics and `ANALYZE`](#54-statistics-and-analyze)
* [6. Write Performance](#6-write-performance)
  * [6.1 Write-Ahead Logging (WAL)](#61-write-ahead-logging-wal)
  * [6.2 MVCC & Vacuuming Overhead](#62-mvcc--vacuuming-overhead)
  * [6.3 Batching and Bulk Writes](#63-batching-and-bulk-writes)
  * [6.4 Write Amplification](#64-write-amplification)
* [7. Replication](#7-replication)
  * [7.1 Physical vs. Logical Replication](#71-physical-vs-logical-replication)
  * [7.2 Asynchronous Replication](#72-asynchronous-replication)
  * [7.3 Synchronous Replication](#73-synchronous-replication)
  * [7.4 Multi-Primary / Active-Active Replication](#74-multi-primary--active-active-replication)
* [8. Data Consistency](#8-data-consistency)
  * [8.1 Transaction Isolation Levels](#81-transaction-isolation-levels)
  * [8.2 Read-After-Write Consistency](#82-read-after-write-consistency)
  * [8.3 Replication Lag & Sticky Routing](#83-replication-lag--sticky-routing)
* [9. Comparative Summary Matrix](#9-comparative-summary-matrix)
* [10. When to Use PostgreSQL (and When Not To)](#10-when-to-use-postgresql-and-when-not-to)
  * [10.1 When to Use PostgreSQL](#101-when-to-use-postgresql)
  * [10.2 When NOT to Use PostgreSQL](#102-when-not-to-use-postgresql)
* [11. When to Consider Alternatives](#11-when-to-consider-alternatives)
  * [11.1 Alternative Database Technologies](#111-alternative-database-technologies)
  * [11.2 Decision Framework Matrix](#112-decision-framework-matrix)

---

## 1. Core Bottlenecks of Single-Node Postgres

### 1.1 CPU and Query Processing Limits
* Parsing complex SQL statements, planning execution paths, and performing heavy compute operations (such as hash joins, sorting, and aggregate functions) consume significant CPU capacity.
* High QPS (queries per second) of unoptimized queries can saturate CPU cores even if the working dataset fits entirely in memory.

### 1.2 Disk I/O and Buffer Pool Cache Misses
* PostgreSQL relies on its internal buffer cache (**Shared Buffers**) and the operating system's page cache to keep hot data pages in RAM.
* When working sets exceed memory capacity, cache misses force the engine to read directly from physical disk (SSD/HDD), causing read latencies to jump from microseconds to milliseconds.

### 1.3 Connection Limits and Overhead
* PostgreSQL operates on a **process-per-connection** backend model rather than a thread-per-connection model.
* Each open client connection consumes memory ($\sim 2\text{--}5\text{ MB}$ base overhead) and increases OS process context-switching overhead. Scaling to thousands of unmanaged direct connections quickly degrades database throughput.

### 1.4 Lock Contention
* Concurrent write operations (`INSERT`, `UPDATE`, `DELETE`) acquire row- or table-level locks.
* Explicit locking patterns (`SELECT ... FOR UPDATE`) or MVCC tuple version bloat can cause read transactions to block behind long-running write transactions or wait on lock allocation.

---

## 2. Read Performance

### 2.1 Memory Architecture & Buffer Pool
* **Shared Buffers:** Memory dedicated to caching database pages (tables and indexes). Configuring this properly (typically $25\%\text{--}40\%$ of total system RAM) reduces physical disk I/O.
* **OS Page Cache:** PostgreSQL relies heavily on the operating system kernel to cache files read from disk, creating a two-tier caching hierarchy.
* **`work_mem`:** Memory allocated for internal sort operations and hash tables per query operation. Setting this too low forces temporary files to spill to disk; setting it too high across concurrent queries causes Out-Of-Memory (OOM) crashes.

### 2.2 Connection Pooling (PgBouncer)
* **Purpose:** Introduces a lightweight proxy layer between application clients and the database to manage active pool sizes.
* **Modes:**
  * **Session Pooling:** Assigns a server connection to a client for the entire connection duration.
  * **Transaction Pooling:** Reassigns server connections per transaction. Ideal for short read-heavy API workloads.
  * **Statement Pooling:** Reassigns connections per SQL statement (disallows multi-statement transactions).

### 2.3 Caching Layers & Strategies
* **Cache-Aside Pattern:** The application reads from an in-memory cache (e.g., Redis). On a cache miss, it queries Postgres, writes the result to cache, and returns it.
* **Cache Stampede / Thundering Herd:** Occurs when a high-traffic key expires, triggering concurrent DB lookups. Mitigate using lock-protected re-computation or early expiration mechanisms (e.g., XFetch).

### 2.4 Denormalization & Materialized Views
* **Denormalization:** Intentionally copying or embedding data into a single table to eliminate multi-table runtime `JOIN` overhead.
* **Materialized Views:** Pre-computed tables created via `CREATE MATERIALIZED VIEW` that store the query output on disk. Must be refreshed explicitly (`REFRESH MATERIALIZED VIEW`) asynchronously or on a scheduled cadence.

---

## 3. Basic Indexing

### 3.1 B-Tree Indexes (The Default)
* **Structure:** Self-balancing search tree maintaining sorted data.
* **Complexity:** Reduces lookups from $O(N)$ full table scans to $O(\log N)$ tree traversals.
* **Supported Operators:** `=`, `<`, `<=`, `>`, `>=`, `BETWEEN`, `IN`, `IS NULL`, and prefix pattern matches (`LIKE 'pattern%'`).

### 3.2 Hash Indexes
* **Structure:** Flat hash table mapping key hashes to tuple IDs.
* **Characteristics:** Optimized exclusively for equality lookups (`=`). Cannot perform range queries or index-assisted sorting.
* **Note:** Fully crash-safe with WAL logging since PostgreSQL 10, but B-Trees remain the default choice.

### 3.3 Multi-Column (Composite) Indexes
* **Column Ordering Rule:** The most selective column or the column used in equality checks should generally precede range query columns.
* **Left-Prefix Rule:** A composite index on `(A, B, C)` can serve queries filtering on `(A)`, `(A, B)`, or `(A, B, C)`, but **cannot** optimize queries filtering solely on `(B)` or `(C)`.

---

## 4. Beyond Basic Indexes

### 4.1 Partial Indexes
* **Concept:** Indexes only a subset of table rows matching a `WHERE` predicate.
* **Syntax:** `CREATE INDEX idx ON orders (created_at) WHERE status = 'PENDING';`
* **Benefits:** Reduces index disk footprint, decreases write maintenance overhead, and speeds up queries targeting high-frequency status flags.

### 4.2 Expression / Functional Indexes
* **Concept:** Indexes the result of a function or scalar expression applied to column values.
* **Syntax:** `CREATE INDEX idx_user_lower_email ON users (LOWER(email));`
* **Use Case:** Speeds up queries using expressions like `WHERE LOWER(email) = 'user@domain.com'`, avoiding table scans caused by function evaluation on every row.

### 4.3 Covering Indexes (`INCLUDE` Clause)
* **Concept:** Attaches payload columns to an index leaf node without adding them to the index search key.
* **Syntax:** `CREATE INDEX idx ON accounts (user_id) INCLUDE (balance);`
* **Benefit (Index-Only Scans):** Allows Postgres to return values (e.g., `balance`) directly from the index tree without fetching the actual table heap page.

### 4.4 Specialized Index Types (GIN, GiST, BRIN)
* **GIN (Generalized Inverted Index):** Optimized for composite types containing multiple values per row (e.g., JSONB keys/values, arrays, full-text search vectors).
* **GiST (Generalized Search Tree):** Used for geometric, spatial (PostGIS), hierarchical, and range data types (e.g., overlaps, bounding box searches).
* **BRIN (Block Range Index):** Stores min/max summaries per block region of disk pages. Extremely small footprint; highly effective for massive append-only naturally ordered tables (e.g., time-series data).

---

## 5. Query Optimization Essentials

### 5.1 `EXPLAIN` and `EXPLAIN ANALYZE`
* **`EXPLAIN`:** Displays the query execution plan generated by the optimizer based on internal database statistics (without executing the query).
* **`EXPLAIN ANALYZE`:** Executes the query and provides actual runtime execution times, total scan costs, memory usage, and row counts alongside the plan estimates.

### 5.2 Understanding Join Strategies
* **Nested Loop Join:** Iterates over the outer table and performs a lookup on the inner table for each row. Ideal for small datasets or indexed joins.
* **Hash Join:** Builds an in-memory hash table from the smaller table, then scans the larger table to find matches. Fast for large un-indexed joins; relies on adequate `work_mem`.
* **Merge Join:** Sorts both datasets on the join key (or uses an index) and scans them concurrently to find matches. Efficient for large sorted inputs.

### 5.3 Avoiding Common Anti-Patterns
* **`SELECT *`:** Retrieves unused columns, increasing network load and preventing Index-Only Scans.
* **Functions on Indexed Columns:** `WHERE UPPER(name) = 'ALICE'` disables standard B-Tree indexes on `name`.
* **Implicit Type Coercion:** Comparing text columns against numeric literals can prevent index usage due to type conversion wrappers.

### 5.4 Statistics and `ANALYZE`
* The Postgres query planner uses sample-based statistics (stored in `pg_statistic` / `pg_stats`) to estimate row counts and cost paths.
* Stale statistics lead to bad execution plans. Run `ANALYZE table_name;` manually after bulk operations, or tune `autovacuum` parameters to update stats automatically.

---

## 6. Write Performance

### 6.1 Write-Ahead Logging (WAL)
* **Mechanism:** Changes are written sequentially to the Write-Ahead Log on disk before modified data pages are written to the main data files.
* **Purpose:** Guarantees durability and atomicity (ACID compliance) while allowing dirty pages in RAM to be written to disk asynchronously (checkpoints).

### 6.2 MVCC & Vacuuming Overhead
* **MVCC (Multi-Version Concurrency Control):** `UPDATE` operations do not modify existing data in place; they insert a new row version (tuple) and mark the old tuple as dead.
* **Vacuuming:** Dead tuples leave storage bloat. The **Autovacuum** daemon reclaims dead tuple space and updates table statistics. Heavy write systems require fine-tuning autovacuum parameters to prevent table bloat and performance degradation.

### 6.3 Batching and Bulk Writes
* **Multi-Row `INSERT`:** Combine single `INSERT` statements into batched values (`INSERT INTO table VALUES (...), (...);`) to lower network round-trips and transaction commit overhead.
* **`COPY` Command:** Use the `COPY` protocol for large-scale data ingestion. It bypasses query parsing overhead and streams data directly into the database.

### 6.4 Write Amplification
* Every active index on a table must be updated whenever an `INSERT`, `DELETE`, or `UPDATE` (affecting indexed columns) occurs.
* Excessively indexing columns slows write throughput and increases disk write activity.

---

## 7. Replication

### 7.1 Physical vs. Logical Replication
* **Physical (Streaming) Replication:** Byte-for-byte stream of the entire WAL disk block changes to replica nodes. Requires identical major Postgres versions and mirrors the whole instance.
* **Logical Replication:** Streams structural row-level change events (e.g., `INSERT`, `UPDATE`, `DELETE`) based on a Publish/Subscribe model. Allows selective table replication, cross-version synchronization, and schema transformations.

### 7.2 Asynchronous Replication
* **Behavior:** The primary node commits writes locally and streams WAL changes to replicas in the background.
* **Trade-off:** High write performance and low primary latency, but introduces potential data loss (RPO $> 0$) if the primary crashes before WAL changes reach replicas.

### 7.3 Synchronous Replication
* **Behavior:** The primary waits for acknowledgment from one or more replicas confirming that WAL data has been written to replica disk before returning success to the client.
* **Trade-off:** Guarantees zero data loss, but write latency equals the round-trip latency to the slowest synchronous replica.

### 7.4 Multi-Primary / Active-Active Replication
* Allows multiple nodes to handle concurrent read and write operations.
* Requires complex distributed conflict detection/resolution mechanisms (e.g., CRDTs or last-write-wins). Rarely built on pure single-node Postgres without third-party extensions like Spock or BDR.

---

## 8. Data Consistency

### 8.1 Transaction Isolation Levels
* **Read Committed (Default):** Prevents dirty reads. Each query inside a transaction sees only data committed before that specific query began.
* **Repeatable Read:** Prevents dirty reads and non-repeatable reads. Queries see a snapshot of data taken at the start of the transaction.
* **Serializable:** Strongest isolation level. Executes transactions concurrently in a manner that guarantees the same result as executing them sequentially, throwing serialization errors on conflict.

### 8.2 Read-After-Write Consistency
* **Issue:** When a user creates or updates a record and immediately reads it back, routing the read query to an asynchronous read replica might yield stale data.
* **Solutions:**
  * Route critical user reads to the **Primary** instance.
  * Use session-aware sticky routing to direct reads to the primary for a set window after writes.

### 8.3 Replication Lag & Sticky Routing
* **Replication Lag:** The microsecond-to-second latency gap between primary commits and replica applications.
* **Sticky Routing:** Directs a user's requests to the Primary node for a set period ($X$ seconds) after perform a write operation, then falls back to replicas once replication is guaranteed to have caught up.

---

## 9. Comparative Summary Matrix

| Strategy | Primary Benefit | Latency Profile | Complexity Added | Key Downsides / Drawbacks |
| :--- | :--- | :--- | :--- | :--- |
| **Indexing** | Eliminates full table scans | Low ($\sim\text{ms}$) | Low | Slows down `INSERT`/`UPDATE` writes; increases disk usage. |
| **Connection Pooling** | Manages server connection overhead | N/A (Throughput tool) | Low | Adds network hop; does not scale storage or compute. |
| **Read Replicas** | Scales read throughput horizontally | Low to Medium | Medium | **Replication lag** causes eventual consistency/stale reads. |
| **Caching Layer** | High-speed read offloading | Very Low ($<\!\text{1ms}$) | High | Cache invalidation risk, stale data, cache stampedes. |
| **Denormalization** | Eliminates multi-table runtime `JOIN`s | Low | Medium to High | Complex write paths, data duplication, risk of drift. |

---

## 10. When to Use PostgreSQL (and When Not To)

### 10.1 When to Use PostgreSQL
* **Structured & Relational Data:** Applications requiring clear schemas, relational joins, foreign keys, and strong data integrity.
* **ACID Transaction Compliance:** Financial systems, e-commerce checkouts, user identity systems, and accounting ledgers requiring strong guarantees.
* **Extensible Multi-Model Workloads:** Systems needing a mix of relational data, JSON document storage (JSONB), geospatial analytics (PostGIS), and vector searching (pgvector) in a single database engines.
* **Moderate to High Scale:** Applications handling terabytes of storage and tens of thousands of QPS when optimized with caching and read replicas.

### 10.2 When NOT to Use PostgreSQL
* **Unbounded Scale (Petabyte-Scale Writes):** Simple key-value access patterns at petabyte scale exceeding single-node write capabilities.
* **High-Frequency Ephemeral Data:** Extremely high-volume transient data like real-time telemetry or cache states that do not need relational guarantees.
* **Extremely High Concurrent Write Ingestion:** Workloads requiring sustained millions of writes per second across distributed geographic regions.

---

## 11. When to Consider Alternatives

### 11.1 Alternative Database Technologies
* **Distributed NoSQL (Cassandra, DynamoDB):** Best for linear write scalability, global active-active replication, and simple key-value/column family patterns without joins.
* **In-Memory Caches (Redis, Memcached):** Best for sub-millisecond key-value retrieval, real-time leaderboards, session stores, and hot caching layers.
* **Distributed OLAP / Data Warehouses (Snowflake, ClickHouse, BigQuery):** Optimized for analytical queries processing billions of rows, column-oriented scans, and heavy ad-hoc aggregations.
* **Distributed SQL (CockroachDB, YugabyteDB):** When relational ACID compliance, SQL support, and multi-region horizontal write scaling are required simultaneously.

### 11.2 Decision Framework Matrix

| Requirement / Workload | Primary Choice | Alternative to Consider | Why? |
| :--- | :--- | :--- | :--- |
| **Core Relational + Transactions** | **PostgreSQL** | MySQL / CockroachDB | Strong ACID guarantees, robust query engine, broad ecosystem. |
| **Sub-millisecond Read Cache** | **Redis** | Memcached | In-memory data structures provide microsecond performance. |
| **Petabyte Write-Heavy Logging** | **Cassandra** | DynamoDB / ClickHouse | Optimized for append-only linear write scaling across nodes. |
| **Analytical Reporting (OLAP)** | **ClickHouse** | Snowflake / BigQuery | Columnar storage engines execute aggregate scans far faster than row-oriented DBs. |
| **Horizontal Multi-Region Writes** | **CockroachDB** | Spanner | Native consensus (Raft/Paxos) distributes writes without single-primary limits. |