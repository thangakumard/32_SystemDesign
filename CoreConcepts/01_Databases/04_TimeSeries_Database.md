# Deep Dive: Time Series Databases Architecture and System Design

## Table of Contents
* **[1. Introduction & Context](#1-introduction--context)**
  * [1.1 Why TSDBs Rare In System Design Interviews](#11-why-tsdbs-rare-in-system-design-interviews)
  * [1.2 Motivating Example: Infrastructure Monitoring at Scale](#12-motivating-example-infrastructure-monitoring-at-scale)
* **[2. The Relational Bottleneck (Why MySQL Fails)](#2-the-relational-bottleneck-why-mysql-fails)**
  * [2.1 Write Throughput Bottleneck](#21-write-throughput-bottleneck)
  * [2.2 Inefficient Storage Utilization](#22-inefficient-storage-utilization)
  * [2.3 High-Latency Range Queries](#23-high-latency-range-queries)
  * [2.4 Relational vs. Time Series Trade-offs](#24-relational-vs-time-series-trade-offs)
* **[3. Core Architectural Building Blocks](#3-core-architectural-building-blocks)**
  * [3.1 Append-Only Storage & Disk I/O Realities](#31-append-only-storage--disk-io-realities)
  * [3.2 Log-Structured Merge-Trees (LSM-Trees)](#32-log-structured-merge-trees-lsm-trees)
  * [3.3 Delta & Delta-of-Deltas Encoding](#33-delta--delta-of-deltas-encoding)
  * [3.4 Query Optimization Metadata: Partitioning, Block-Level Summaries & Bloom Filters](#34-query-optimization-metadata-partitioning-block-level-summaries--bloom-filters)
  * [3.5 Downsampling and Retention Policies](#35-downsampling-and-retention-policies)
* **[4. End-to-End Execution Flow of a TSDB](#4-end-to-end-execution-flow-of-a-tsdb)**
  * [4.1 Data Modeling: Metrics, Tags, Fields, and Timestamps](#41-data-modeling-metrics-tags-fields-and-timestamps)
  * [4.2 The Write Path](#42-the-write-path)
  * [4.3 The Read Path](#43-the-read-path)
* **[5. The High-Cardinality Trap](#5-the-high-cardinality-trap)**
  * [5.1 Why High Cardinality Breaks Memtables and Indexing](#51-why-high-cardinality-breaks-memtables-and-indexing)
  * [5.2 Anti-Pattern: Using TSDB for General Event Tracking](#52-anti-pattern-using-tsdb-for-general-event-tracking)
* **[6. Summary & System Design Takeaways](#6-summary--system-design-takeaways)**

---

## 1. Introduction & Context

Time Series Databases (TSDBs) are hyper-specialized storage systems engineered to accommodate highly specific write and read access patterns. Unlike traditional general-purpose relational or document databases, TSDBs operate on sequences of values indexed in time order.

### 1.1 Why TSDBs Rare In System Design Interviews

Despite their extensive production use (e.g., Prometheus, InfluxDB, TimescaleDB), TSDBs are rarely prescribed as primary solutions in system design interviews for two core reasons:
1. **Non-uniform Familiarity:** Interviewers themselves vary widely in their deep technical understanding of specialized TSDB engines.
2. **Narrow Scope of Utility:** TSDBs excel under precise assumptions. Applying them to scenarios outside these assumptions leads to degraded performance.

However, studying TSDB mechanics yields fundamental distributed systems primitives—such as append-only storage, LSM-trees, delta encoding, and probabilistic indexing—that enhance your overall system design toolkit.

### 1.2 Motivating Example: Infrastructure Monitoring at Scale

Consider maintaining a server fleet for a high-scale tech company (e.g., Airbnb, Meta):
* **Fleet Size:** $N = 100,000$ servers.
* **Metrics Tracked per Server:** CPU Utilization, Memory Usage, Disk Input, Disk Output, Network Egress ($M = 5$).
* **Reporting Frequency:** Every $T = 10$ seconds.
* **Consumer Queries:**
  1. *Operations (Real-time):* "What is the average CPU load on my web servers over the past 5 minutes?"
  2. *Alerting (Continuous High-Frequency):* "Has any web server exceeded 80% CPU utilization in the last 30 seconds?"
  3. *Capacity Planning (Batch Low-Frequency):* "What is the projected compute growth for the next fiscal year based on 12-month trends?"

---

## 2. The Relational Bottleneck (Why MySQL Fails)

Suppose a standard relational database (e.g., MySQL) is deployed to store this metric stream using a canonical schema.

### Relational Database Naive Schema (`metrics` table)

| Column Name | Data Type | Size (Bytes) | Description |
| :--- | :--- | :--- | :--- |
| `timestamp` | `DATETIME` / `BIGINT` | 8 | Observation timestamp |
| `host` | `VARCHAR(255)` | ~32–256 | Unique hostname string |
| `metric_name` | `VARCHAR(64)` | ~8–64 | Metric type (`cpu`, `memory`, etc.) |
| `value` | `DOUBLE` | 8 | Floating-point metric observation |

```sql
CREATE TABLE metrics (
    timestamp BIGINT NOT NULL,
    host VARCHAR(255) NOT NULL,
    metric_name VARCHAR(64) NOT NULL,
    value DOUBLE NOT NULL,
    INDEX idx_lookup (metric_name, timestamp)
);
```

### 2.1 Write Throughput Bottleneck

Calculating write throughput:
* **Average Writes Per Second:**
  $$\text{RPS}_{\text{avg}} = \frac{N \times M}{T} = \frac{100,000 \times 5}{10} = 50,000 \text{ writes/sec}$$
* **Worst-Case Peak (Thundering Herd):** If reporting agents synchronize, all 100,000 nodes send metrics in the same 1-second window:
  $$\text{RPS}_{\text{peak}} = 100,000 \times 5 = 500,000 \text{ writes/sec}$$

Single-instance relational databases generally bottleneck around 10,000 write operations/sec due to lock contention, transaction logging overhead, and B-Tree index updates requiring random page writes. Sharding MySQL across metric types or hosts adds operational complexity and cost without addressing fundamental storage inefficiencies.

### 2.2 Inefficient Storage Utilization

Every row incurs significant overhead:
* Uncompressed timestamp: 8 bytes
* String fields (`host`, `metric_name`): 40+ bytes (unless normalized into lookup tables)
* Double-precision payload: 8 bytes
* Index entries (e.g., B-Tree overhead): ~32+ bytes per row

At 50,000 writes/sec, the system generates:
$$\text{Rows per Day} = 50,000 \times 86,400 = 4.32 \times 10^9 \text{ rows/day}$$
Storing $4.32 \text{ billion rows/day}$ on disk without specialization rapidly consumes terabytes of storage unnecessarily.

### 2.3 High-Latency Range Queries

Executing an analytical query over thousands of web servers requires scanning extensive index structures:

```sql
SELECT AVG(value) 
FROM metrics 
WHERE metric_name = 'cpu' 
  AND timestamp >= 1700000000 
  AND timestamp < 1700028800;
```

Because B-Tree nodes store data in fixed-size pages optimized for point lookups, evaluating billions of rows forces excessive disk page reads into memory. Queries take minutes instead of milliseconds, competing directly with incoming write streams for I/O bandwidth.

### 2.4 Relational vs. Time Series Trade-offs

| Factor | Traditional RDBMS (e.g., MySQL) | Time Series Database (TSDB) |
| :--- | :--- | :--- |
| **Primary Indexing** | B-Tree / B+Tree (optimized for point lookups) | Time-bucketed / Append-Only (LSM / Segmented) |
| **Write Pattern** | In-place updates, random disk allocations | Append-only sequential streams |
| **Compression** | Row-level or page-level general block compression | Domain-specific (Delta-of-Deltas, XOR/Gorilla) |
| **Data Lifecycle** | Deletions via row-level `DELETE` (creates fragmentation) | Drop whole time partitions/blocks (zero-fragmentation) |
| **High Cardinality** | Supported via arbitrary B-Trees | Breaks inverted tag indexes |

---

## 3. Core Architectural Building Blocks

To resolve the throughput, storage, and latency constraints, TSDBs combine several targeted architectural patterns.

### 3.1 Append-Only Storage & Disk I/O Realities

Physical disk drives (HDDs) utilize spinning platters and mechanical actuator arms:
* **Disk Seek Time:** Moving the read/write head to a specific track requires physical mechanical movement (~5–10 ms per seek).
* **Seek Capacity:** HDDs cap around 100 to 200 random seek operations per second.
* **Solid State Drives (SSDs):** SSDs eliminate mechanical components, but underlying flash memory block structures still read and program contiguous blocks faster than performing random small rewrites (which trigger high garbage collection and wear leveling overhead).

```
Random Writes (Inefficient - Requires repeated disk seeks):
[Sector A: Write 1] ---> [Sector X: Write 2] ---> [Sector C: Write 3]

Sequential / Append-Only Writes (Optimal - Single sweep):
[Block 1: Write 1][Block 2: Write 2][Block 3: Write 3][Block 4: Write 4]...
```

By constraining write operations exclusively to **append-only append logs**, write throughput approaches the theoretical maximum sequential write bandwidth of the underlying hardware media.

### 3.2 Log-Structured Merge-Trees (LSM-Trees)

To maintain sorted data structures on disk without performing in-place random writes, TSDBs employ Log-Structured Merge-Trees (LSM-Trees).

```
  Incoming Writes 
        │
        ├───► [ Write-Ahead Log (WAL) ] (Persisted directly to Disk for durability)
        │
        └───► [ Memtable ] (In-Memory sorted buffer e.g., SkipList/Red-Black Tree)
                    │
                    │ (Flush when limit reached)
                    ▼
              [ SSTable / Segment File (Disk) ]  <-- Immutable Sequential File
              [ SSTable / Segment File (Disk) ]  <-- Immutable Sequential File
                    │
                    │ (Background Thread)
                    ▼
              [ Compaction Process ]  ──► Merges adjacent segments into unified larger blocks
```

1. **Write-Ahead Log (WAL):** Incoming writes append directly to a sequential WAL file on disk. This step ensures crash recovery.
2. **Memtable:** Simultaneously, data updates an in-memory sorted structure (e.g., SkipList or Red-Black Tree). The memtable accumulates and sorts metrics by timestamp and tag combination.
3. **Flushing to Immutable Files:** Once the memtable reaches a configured byte size or time interval, it flushes to disk as an immutable Immutable Segment/SSTable.
4. **Compaction:** Offline background tasks periodically merge overlapping immutable segments, discarding duplicate or tombstoned entries to optimize long-term read latency.

### 3.3 Delta & Delta-of-Deltas Encoding

Because time series observations are typically regularly sampled numbers, values fluctuate gradually over short windows. Rather than writing raw 64-bit values, TSDBs store differences between consecutive values.

#### 1. Delta Encoding for Values
Given a series of temperature readings: $V = [56.7, 58.4, 58.9, 58.2]$
* $V_0 = 56.7$ (Stored as initial base value)
* $\Delta_1 = 58.4 - 56.7 = +1.7$
* $\Delta_2 = 58.9 - 58.4 = +0.5$
* $\Delta_3 = 58.2 - 58.9 = -0.7$

Storing small floating-point offsets requires fewer bits than storing raw IEEE-754 64-bit double-precision representations.

#### 2. Delta-of-Deltas Encoding for Timestamps
Given fixed 10-second sampling intervals:
* Absolute Timestamps: $T = [1700000000, 1700000010, 1700000020, 1700000030]$
* First Delta ($\Delta t = t_n - t_{n-1}$): $[10, 10, 10, 10]$
* Delta-of-Deltas $D = \Delta t_n - \Delta t_{n-1}$:
  * $D_0 = 10$ (Initial step)
  * $D_1 = 10 - 10 = 0$
  * $D_2 = 10 - 10 = 0$
  * $D_3 = 10 - 10 = 0$

Streams of zeroes are heavily compressible using Bit-Packing or Variable-Byte Encoding. As shown in Facebook's *Gorilla* TSDB paper, average timestamp storage costs drop from **32 bits to roughly 1.37 bits per data point**.

### 3.4 Query Optimization Metadata: Partitioning, Block-Level Summaries & Bloom Filters

To prevent reading every segment file during execution, TSDBs construct block metadata:

```
+--------------------------------------------------------------------------------+
| Segment Block File (e.g., 2026-11-07 10:00:00 to 12:00:00)                    |
+------------------------------------------------------+-------------------------+
| Block Header / Metadata Summary                      | Compressed Payload Data |
| - Time Range: [1700000000, 1700007200]              |                         |
| - Value Stats: Min = 12.1, Max = 94.5, Sum = 4501.2  | [ Series 1 Data ]       |
| - Hosts Bloom Filter: [01101001...]                  | [ Series 2 Data ]       |
+------------------------------------------------------+-------------------------+
```

1. **Time-Based Partitioning:** Data files are split into bounded temporal windows (e.g., 2-hour or 24-hour chunks). Queries bounded within a time range prune irrelevant blocks immediately.
2. **Block Summary Metadata (Min/Max/Sum):** Each segment block records local aggregations. If a query requests max CPU utilization, but a segment's local `Max` is lower than an established threshold, the engine skips processing the block payload entirely.
3. **Bloom Filters:** A space-efficient probabilistic data structure used to check set membership. Before opening an immutable file to locate host `web-server-99`, the query engine evaluates the block's Bloom filter:
   * *Returns `False`:* The host is **definitely not** in the file (skip disk read).
   * *Returns `True`:* The host **might** be in the file (read and evaluate index).

### 3.5 Downsampling and Retention Policies

High-resolution metric data (e.g., 1-second cadence) retains its maximum utility within immediate operational windows. Historical data rarely requires sub-second precision.

```
Raw Telemetry (10s resolution) ──► Retained for 7 Days  ──► Full Detail
                                           │
                                           ▼ (Background Downsampling)
1-Minute Aggregations (Avg/Max/Min) ──► Retained for 30 Days ──► Medium Detail
                                           │
                                           ▼ (Background Downsampling)
1-Hour Aggregations (Avg/Max/Min)  ──► Retained for 1 Year  ──► Low Detail
```

TSDB background routines sweep old immutable blocks, aggregate point samples via interpolation or statistical summaries (Average, Min, Max, Count), write out new downsampled segment files, and purge raw granular segments to reclaim disk capacity.

---

## 4. End-to-End Execution Flow of a TSDB

### 4.1 Data Modeling: Metrics, Tags, Fields, and Timestamps

TSDBs format data into structured observations:
* **Metric Name:** The physical entity tracked (e.g., `system.cpu.user`).
* **Tags (Labels):** Key-value metadata strings identifying dimensions (e.g., `host=web-12`, `region=us-west`, `env=prod`).
* **Fields:** Actual scalar payload values (e.g., `usage=45.2`, `idle=54.8`).
* **Timestamp:** Epoch marker (e.g., `1700000000`).

A single unique combination of a **Metric Name + Tags** defines a **Series Key (Time Series)**:
$$\text{Series Key} = \text{\texttt{system.cpu.user}}\{\text{\texttt{env}}=\text{\texttt{prod}}, \text{\texttt{host}}=\text{\texttt{web-12}}, \text{\texttt{region}}=\text{\texttt{us-west}}\}$$

### Sample Ingestion Schema Representation

| Metric | Tags (Inverted Index Keys) | Fields (Values) | Timestamp |
| :--- | :--- | :--- | :--- |
| `cpu_usage` | `host=web-01`, `region=us-west`, `env=prod` | `val=82.1` | `1700000000` |
| `cpu_usage` | `host=web-02`, `region=us-west`, `env=prod` | `val=14.5` | `1700000000` |
| `cpu_usage` | `host=web-01`, `region=us-east`, `env=stage` | `val=43.0` | `1700000000` |

### 4.2 The Write Path

```
Incoming Metric Stream
        │
        ▼
1. Append to Write-Ahead Log (WAL) ──► Disk Storage (For recovery safety)
        │
        ▼
2. Ingest into Memtable (In-Memory Buffer)
        │
        ├─► Map (Metric + Tags) -> Series ID
        └─► Sort entries chronologically per Series ID
        │
        ▼
3. Threshold Triggered (Size/Time) -> Flush Memtable to Disk Segment File
        │
        ├─► Compress Timestamps via Delta-of-Deltas Encoding
        ├─► Compress Values via Delta/XOR Encoding
        ├─► Generate Block-Level Metadata & Bloom Filters
        └─► Write Immutable Segment File + Inverted Index
```

### 4.3 The Read Path

Consider a metric lookup: 
`Find average CPU usage for env=prod, region=us-west between T1 and T2`

```
Query Request: AVG(cpu_usage) WHERE region='us-west' AND env='prod' FROM T1 to T2
        │
        ▼
1. Time-Range Pruning
        └─► Evaluate segment headers. Select only segment files containing range [T1, T2].
        │
        ▼
2. Index Search & Series Matching
        └─► Query Inverted Index for tags (region='us-west' AND env='prod').
        └─► Retrieve target Series IDs (e.g., Series ID #102, #108).
        │
        ▼
3. Block Filtering & Loading
        └─► Run Bloom Filters and Block Min/Max stats to eliminate non-matching blocks.
        └─► Read matching compressed data blocks sequentially from disk.
        │
        ▼
4. Decompression & Post-Processing
        └─► Decode Delta-of-Deltas timestamps and XOR-encoded value streams.
        └─► Calculate average aggregation over targeted series across time range [T1, T2].
        └─► Return aggregated vector to client.
```

---

## 5. The High-Cardinality Trap

### 5.1 Why High Cardinality Breaks Memtables and Indexing

**Cardinality** refers to the number of unique time series managed by the database. 
$$\text{Total Series} = \prod \text{Unique Tag Values}$$

If tags include low-cardinality values:
* `environment` (2 values: `prod`, `stage`)
* `region` (5 values: `us-east-1`, `us-west-1`, etc.)
* Total Series = $2 \times 5 = 10$ series.

However, if an engineer adds a high-cardinality tag like `user_id` ($100,000,000$ unique users) or `request_id` ($1,000,000,000$ unique UUIDs):
* Total Series = $2 \times 5 \times 1,000,000,000 = 10,000,000,000$ unique time series.

```
Low-Cardinality Series Layout (Efficient):
Series #1: [T1, T2, T3, T4, T5, T6, T7, T8, ...] (Long contiguous memory blocks)
Series #2: [T1, T2, T3, T4, T5, T6, T7, T8, ...]

High-Cardinality Series Layout (Broken):
Series #1: [T1]
Series #2: [T1]
...
Series #10,000,000: [T1] (Trillions of micro-buffers; destroys Memtable overhead)
```

High cardinality causes systemic failures in TSDB engines:
1. **Memtable Saturation:** The TSDB keeps dedicated memory buffers and index tracks for every distinct series. Millions of active series consume available RAM for index pointers, leaving no room for actual data point buffering.
2. **Inverted Index Inflation:** Tag lookup tables balloon in size. Querying the index to locate matching series ID postings lists degrades from $O(1)$ lookups to multi-gigabyte memory scans.
3. **Loss of Delta Compression Efficiency:** Delta-of-Deltas compression relies on long sequences of contiguous measurements for a single series. With unique IDs, series only contain 1 or 2 observations, invalidating compression mechanisms and driving storage requirements back to uncompressed formats.

### 5.2 Anti-Pattern: Using TSDB for General Event Tracking

#### Problematic Schema: Post Like Events
An engineer attempts to track user likes on posts via a TSDB:

```
# ANTI-PATTERN: High-cardinality tag set
Metric: post.like
Tags: user_id=892311, post_id=9928131, region=us-west
Value: 1
Timestamp: 1700000000
```

#### Failure Analysis
Because `user_id` and `post_id` have unbounded uniqueness, almost every incoming point registers a brand-new series. The database spends nearly all its memory allocating inverted index pointers and flushed small disk blocks containing single data points.

#### Alternative Solutions
For high-cardinality transaction or user-event tracking, use appropriate storage primitives:
* **Relational / OLAP Databases (e.g., ClickHouse, PostgreSQL, Snowflake):** Optimized for columnar scans over large row counts with high-cardinality fields.
* **Key-Value / Cache Stores (e.g., Redis):** For real-time increment counters (`INCRBY`).

---

## 6. Summary & System Design Takeaways

### Core Principles Summary

```
                      ┌─────────────────────────────────────────┐
                      │    Time Series Database Architecture    │
                      └────────────────────┬────────────────────┘
                                           │
         ┌─────────────────────────────────┼─────────────────────────────────┐
         ▼                                 ▼                                 ▼
┌─────────────────┐               ┌─────────────────┐               ┌─────────────────┐
│ High Write-Rate │               │ Storage Scale   │               │ Query Velocity  │
├─────────────────┤               ├─────────────────┤               ├─────────────────┤
│ • Append-Only   │               │ • Delta Encoding│               │ • Partitioning  │
│ • LSM-Trees     │               │ • Delta-of-Delta│               │ • Bloom Filters │
│ • WAL + MemTable│               │ • Downsampling  │               │ • Block Stats   │
└─────────────────┘               └─────────────────┘               └─────────────────┘
```

1. **Avoid Random Disk Writes:** Leverage LSM-trees, WALs, and memtables to transform incoming random data flushes into contiguous, append-only disk operations.
2. **Exploit Domain Structures for Compression:** Leverage delta and delta-of-deltas encoding (e.g., Gorilla) to store timestamps and gradually changing numeric streams using bit-level offsets.
3. **Prune Query Paths Early:** Implement block metadata (time boundaries, min/max statistics) and Bloom filters to avoid reading irrelevant segment files during execution.
4. **Mind the Cardinality Bound:** Ensure tag keys maintain constrained value spaces. High-cardinality tag sets destroy TSDB indexing, compression, and RAM footprints. Use columnar OLAP engines (e.g., ClickHouse) when querying across high-cardinality attributes.