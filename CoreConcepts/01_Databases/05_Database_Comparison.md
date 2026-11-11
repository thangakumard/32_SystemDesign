# Database Comparison: Throughput, Architecture & System Design Trade-offs

> *A side-by-side comparison of the database families covered in this folder — PostgreSQL, MySQL, DynamoDB, Cassandra, Vector Databases, and Time Series Databases — with representative maximum read/write RPS figures for system design interview reasoning.*

---

## Table of Contents
1. [Why RPS Numbers Are "It Depends"](#1-why-rps-numbers-are-it-depends)
2. [Quick Comparison Table (Max Read RPS vs. Max Write RPS)](#2-quick-comparison-table-max-read-rps-vs-max-write-rps)
3. [Per-Database Breakdown](#3-per-database-breakdown)
4. [Architecture & Scaling Model Comparison](#4-architecture--scaling-model-comparison)
5. [Consistency & Durability Comparison](#5-consistency--durability-comparison)
6. [Decision Guide: Picking by Access Pattern](#6-decision-guide-picking-by-access-pattern)

---

## 1. Why RPS Numbers Are "It Depends"

Throughput numbers below are **order-of-magnitude, single-node (or single-partition) reference points** drawn from common system design interview framing and published benchmarks (sysbench, YCSB, AWS documented limits). Real numbers swing 10x+ based on:

* Row/item size, index count, and query complexity (point lookup vs. join/aggregation).
* Hardware (NVMe vs. HDD, core count, RAM for buffer cache).
* Consistency level chosen (eventual vs. strong; `ONE` vs. `QUORUM` vs. `ALL`).
* Whether the number is **per-node/per-partition** (vertical limit) or **cluster-wide** (horizontal, which can scale ~linearly by adding nodes/partitions).

Use these as **relative intuition** ("Cassandra/DynamoDB write path is write-optimized and scales horizontally without bound; PostgreSQL/MySQL write path is vertically bottlenecked on a single primary") rather than exact SLAs.

---

## 2. Quick Comparison Table (Max Read RPS vs. Max Write RPS)

| Database | Max Read RPS (single node / partition) | Max Write RPS (single node / partition) | Horizontal Scale-Out | Notes |
| :--- | :--- | :--- | :--- | :--- |
| **PostgreSQL** | ~50,000–100,000 QPS (cache-hit, indexed point reads) | ~10,000–20,000 TPS (single primary, WAL-bound) | Reads: near-linear via read replicas. Writes: vertical only (single primary) unless sharded (Citus/CockroachDB-style) | Writes bottleneck on WAL fsync + lock contention + index maintenance; see [`Postgres/postgresql_read_scaling_cheat_sheet.md`](Postgres/postgresql_read_scaling_cheat_sheet.md) |
| **MySQL (InnoDB)** | ~50,000–150,000 QPS (cache-hit, point reads via clustered index) | ~10,000–50,000 TPS (single primary, binlog-bound) | Reads: linear via read replicas. Writes: vertical only unless sharded (Vitess) | Similar B+Tree/WAL-style bottlenecks as PostgreSQL; marginally higher raw QPS ceiling on simple PK lookups due to a lighter-weight optimizer |
| **Amazon DynamoDB** | ~3,000 reads/sec per partition (eventually consistent); ~1,500/sec strongly consistent | ~1,000 writes/sec per partition | **Unbounded** — add partitions (via partition key design) to scale linearly; AWS fleet-wide handles tens of millions RPS in aggregate | Per-partition limit is the real ceiling; hot partition = throttling regardless of table-level provisioned/on-demand capacity |
| **Apache Cassandra** | ~5,000–15,000 reads/sec per node (consistency level dependent) | ~10,000–20,000+ writes/sec per node (append-only, LSM-optimized) | **Linear** — add nodes to scale both reads and writes; production clusters handle 100,000s–millions of writes/sec aggregate | Write path is intentionally faster than read path (sequential append vs. multi-SSTable merge) |
| **Vector Database** (HNSW-based, e.g., Milvus/Pinecone/pgvector) | ~1,000–5,000 ANN queries/sec per node (recall/dimension dependent) | ~1,000–10,000 inserts/sec per node (graph-insert bound) | Reads: scale via sharded/replicated index segments. Writes: bound by hot/cold tiering + periodic index rebuild | HNSW insert is $O(\log N)$ graph traversal — much slower than read-optimized traversal; see [`03_Vector_Databases.md`](03_Vector_Databases.md) §6 |
| **Time Series Database** (e.g., InfluxDB/Prometheus/TimescaleDB-style) | ~10,000–50,000 range-query RPS per node (time-bounded, block-pruned) | ~100,000–500,000 points/sec per node (append-only, LSM + delta encoding) | Writes: near-linear via sharding by series/time. Reads: linear via replica + downsampled tiers | Write-optimized by design (sequential append + compression); see [`04_TimeSeries_Database.md`](04_TimeSeries_Database.md) §2.1 |

> **Headline takeaway:** PostgreSQL/MySQL are **read-friendly, write-constrained** single-primary systems. DynamoDB/Cassandra are **write-optimized and horizontally unbounded** by design. Vector DBs are **read-optimized, insert-slow** (graph/cluster maintenance cost). Time Series DBs are **write-optimized above all else** (sequential append + compression), with read latency protected by time-based pruning.

---

## 3. Per-Database Breakdown

### PostgreSQL
* **Bottleneck:** Single-primary WAL writes, MVCC tuple bloat, lock contention on hot rows, connection overhead (process-per-connection).
* **Read scaling lever:** Shared buffers + OS page cache + PgBouncer pooling + read replicas (horizontal, near-linear).
* **Write scaling lever:** Batching/`COPY`, reducing index count, vertical scaling, or sharding (Citus) as a last resort.
* **Reference:** [`Postgres/postgresql_read_scaling_cheat_sheet.md`](Postgres/postgresql_read_scaling_cheat_sheet.md)

### MySQL (InnoDB)
* **Bottleneck:** Same family of constraints as PostgreSQL — binlog-bound single-primary writes, buffer pool cache misses, row/table locks.
* **Read scaling lever:** Read replicas (async replication), ProxySQL/connection pooling.
* **Write scaling lever:** Group Replication / semi-sync for durability, or Vitess for sharding at scale (used by YouTube, Slack).
* **Why it's grouped with Postgres:** Both are single-leader B-Tree relational engines with nearly identical throughput ceilings in practice — the real-world difference is usually operational/ecosystem, not raw RPS.

### Amazon DynamoDB
* **Bottleneck:** Per-partition throughput ceiling (3,000 RCU / 1,000 WCU), not the table as a whole — a bad partition key creates a "hot partition" that throttles regardless of overall provisioned capacity.
* **Read scaling lever:** DAX (microsecond cache), eventually-consistent reads (half the RCU cost), GSIs to re-shape access patterns.
* **Write scaling lever:** Good partition key cardinality so writes spread evenly across many partitions — this is how it scales to effectively unbounded aggregate throughput.
* **Reference:** [`01_DynamoDB.md`](01_DynamoDB.md) §5

### Apache Cassandra
* **Bottleneck:** Read path merges Memtable + multiple SSTables (bloom filter + disk seeks); write path is a pure sequential append (CommitLog + Memtable), which is why writes consistently outperform reads per-node.
* **Read scaling lever:** Lower consistency level (`ONE` vs `QUORUM`), compaction tuning to reduce SSTable fragmentation.
* **Write scaling lever:** Add nodes — write throughput scales linearly because there's no single leader to bottleneck on.
* **Reference:** [`02_Cassandra.md`](02_Cassandra.md) §6, §8 ("Green Flags": write volume > 100k/sec)

### Vector Databases
* **Bottleneck:** HNSW graph inserts require $O(\log N)$ traversals to link neighbors across layers — this is inherently slower than read traversal, which is why hot/cold tiering (buffer unindexed writes, periodically rebuild the index) is the standard architecture.
* **Read scaling lever:** Tune `recall` target down, shard the index, add replicas for the cold (read-only) index.
* **Write scaling lever:** Batch inserts into the hot buffer; schedule async rebuilds into the cold index rather than updating it in place.
* **Reference:** [`03_Vector_Databases.md`](03_Vector_Databases.md) §4, §6

### Time Series Databases
* **Bottleneck:** High-cardinality tags (e.g., accidentally indexing `user_id`) destroy the memtable/inverted-index efficiency this architecture depends on — see the cardinality trap.
* **Read scaling lever:** Time-range pruning, block-level min/max summary skipping, bloom filters on tags, downsampled tiers for older data.
* **Write scaling lever:** This is the native strength — append-only LSM writes plus delta/delta-of-deltas encoding reach the highest raw write RPS of any database family in this comparison.
* **Reference:** [`04_TimeSeries_Database.md`](04_TimeSeries_Database.md) §2.1, §5

---

## 4. Architecture & Scaling Model Comparison

| Database | Data Model | Write Path Architecture | Default Scaling Direction |
| :--- | :--- | :--- | :--- |
| PostgreSQL | Relational (rows, B-Tree) | In-place update + WAL | Vertical (primary), horizontal for reads only |
| MySQL | Relational (rows, B+Tree, InnoDB) | In-place update + binlog/redo log | Vertical (primary), horizontal for reads only |
| DynamoDB | Key-Value / Document | Leader node + async quorum replication | Horizontal (auto-partitioned) |
| Cassandra | Wide-Column | Append-only (CommitLog → Memtable → SSTable) | Horizontal (leaderless, linear) |
| Vector DB | Embeddings + metadata | Hot buffer + cold immutable ANN index (HNSW/IVF) | Horizontal (sharded index segments) |
| Time Series DB | Metric + tags + timestamp | Append-only (WAL → Memtable → compressed segment) | Horizontal (shard by series/time) |

---

## 5. Consistency & Durability Comparison

| Database | Default Consistency | Tunable? | Durability Mechanism |
| :--- | :--- | :--- | :--- |
| PostgreSQL | Strong (single primary, Read Committed) | Yes — isolation levels (Read Committed/Repeatable Read/Serializable) | WAL + fsync |
| MySQL | Strong (single primary) | Yes — isolation levels; semi-sync replication | Binlog + redo log |
| DynamoDB | Eventual (default) | Yes — eventually consistent vs. strongly consistent reads | Quorum-replicated writes across AZs |
| Cassandra | Eventual (AP system) | Yes — per-query consistency level (`ONE`/`QUORUM`/`ALL`) | CommitLog (sequential fsync) |
| Vector DB | Eventual (hot/cold tiers) | Limited — depends on engine | Varies; many treat the vector index as a derived/rebuildable artifact, not system of record |
| Time Series DB | Eventual/near-real-time | Limited | WAL + periodic segment flush |

---

## 6. Decision Guide: Picking by Access Pattern

* **Need ACID transactions + relational joins, moderate scale (10K–50K QPS):** PostgreSQL or MySQL.
* **Need unbounded horizontal write scale, known key-based access patterns, single-digit-ms latency:** DynamoDB.
* **Need the highest raw write throughput (>100K writes/sec), predictable query patterns, no joins:** Cassandra.
* **Need semantic/similarity search over embeddings:** Vector database (pgvector if <10M vectors and already on Postgres; Milvus/Pinecone/Qdrant beyond that).
* **Need to ingest and query high-frequency metrics (IoT, infra monitoring) with low-cardinality tags:** Time Series Database.
* **Anti-pattern reminder:** None of the NoSQL options here replace PostgreSQL/MySQL for workloads needing ad-hoc joins, strong relational integrity, or low write volume — the extra operational complexity isn't worth it below roughly 10K sustained writes/sec.
