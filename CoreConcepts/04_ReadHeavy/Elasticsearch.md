# Elasticsearch System Design & Architecture Guide

## Table of Contents

1. [Part 1: Elasticsearch from a System Design Perspective](#part-1-elasticsearch-from-system-design-perspective)
   * [What is Elasticsearch?](#what-is-elasticsearch)
   * [When Should It Be Used?](#when-should-it-be-used)
   * [When Should It NOT Be Used?](#when-should-it-not-be-used)

2. [Part 2: Must-Know Elasticsearch Internals & Features for a Senior Engineer](#part-2-must-know-elasticsearch-internals--features-for-a-senior-engineer)
   * [Inverted Index & Tokenization](#inverted-index--tokenization)
   * [Shards & Replicas](#shards--replicas)
   * [Dynamic vs. Explicit Mappings](#dynamic-vs-explicit-mappings)
   * [Text vs. Keyword Datatypes](#text-vs-keyword-datatypes)
   * [BM25 Scoring Algorithm](#bm25-scoring-algorithm)
   * [Geo-Spatial Capabilities & `geo_distance`](#geo-spatial-capabilities--geo_distance)
   * [Aggregations & Analytical Engine](#aggregations--analytical-engine)
   * [Refresh Interval & Index Lifecycle Management (ILM)](#refresh-interval--index-lifecycle-management-ilm)
   * [Routing Keys](#routing-keys)

3. [Part 3: Pros and Cons of Elasticsearch](#part-3-pros-and-cons-of-elasticsearch)
   * [Pros](#pros)
   * [Cons](#cons)

## Part 1: Elasticsearch from a System Design Perspective

### What is Elasticsearch?
Elasticsearch is a distributed, RESTful search and analytics engine built on top of Apache Lucene. In system design, it is rarely deployed as a primary database of record; instead, it acts as a **polyglot persistence layer, secondary index, and analytical datastore**.

### When Should It Be Used?
You should reach for Elasticsearch when your architecture demands capabilities that traditional relational databases (RDBMS) or standard NoSQL document stores (like MongoDB) handle poorly:

1. **Complex Full-Text Search:** Users need typo-tolerance, stemming, synonyms, language-specific text processing, and relevance scoring (BM25) across millions of documents.
2. **Log Aggregation & Observability (SIEM):** Ingesting terabytes of streaming logs, metrics, or security events per day with fast time-range filtering and aggregations (the core of the ELK/Elastic Stack).
3. **Geo-Spatial Queries:** Finding entities "within $X$ radius of current location" coupled with structured filters at scale.
4. **Hybrid / Vector Search (AI/RAG Workloads):** Storing dense vector embeddings alongside traditional metadata to perform semantic search or power LLM Retrieval-Augmented Generation (RAG).

### When Should It NOT Be Used?
* **As a Primary Source of Truth for OLTP:** Elasticsearch prioritizes search speed and horizontal scale over strict ACID compliance. While modern versions feature improved concurrency controls (MVCC), it lacks robust transactional rollback guarantees found in Postgres or MySQL.
* **Heavy Relational Joins:** While parent/child relationships and nested documents exist, executing heavy runtime joins across multiple collections destroys query latency. Data should be pre-joined/denormalized before ingestion.

## Part 2: Must-Know Elasticsearch Internals & Features for a Senior Engineer

To design scalable systems with Elasticsearch, you must master how it organizes data under the hood and how to tune its performance, including its core search and analytical mechanisms.

### Inverted Index & Tokenization
The foundational data structure. Instead of mapping rows to values, it maps unique words (tokens) to the documents containing them, enabling $O(1)$ or $O(\log N)$ text lookups. Analyzers break text down using tokenizers and filters before indexing.

### Shards & Replicas
An index is broken into **Primary Shards** (for write distribution) and **Replica Shards** (for read scaling and fault tolerance). *Senior rule of thumb:* Keep individual shard sizes between **10GB to 50GB** to avoid garbage collection and heap pressure issues.

### Dynamic vs. Explicit Mappings
Letting Elasticsearch dynamically guess types (e.g., turning strings into text *and* keyword fields) bloats memory and causes mapping explosions. Production systems **always enforce explicit mapping templates**.

### Text vs. Keyword Datatypes
* **`text`** fields are analyzed (tokenized, lowercased) for full-text search.
* **`keyword`** fields are exact-match values, ideal for filtering, sorting, and aggregations (e.g., status codes, user IDs).

### BM25 Scoring Algorithm
Elasticsearch uses **Okapi BM25** (Best Matching 25) as its default relevance scoring function (evolving from older TF-IDF implementations). 
* **How it works:** It calculates how well a document matches a query based on term frequency (TF), inverse document frequency (IDF), and **document length normalization** (preventing excessively long documents from unfairly dominating scores simply because they contain more words).
* **System Design Impact:** Tuning parameters like $k_1$ (controls term frequency saturation) and $b$ (controls the degree of document length penalty) allows senior engineers to optimize search precision versus recall for unique domain text.

### Geo-Spatial Capabilities & `geo_distance`
Elasticsearch provides high-performance location-based querying using fields mapped as `geo_point` (latitude/longitude pairs) or `geo_shape` (polygons/complex geometries).
* **`geo_distance` Query:** Allows you to filter or sort documents within a precise radius of a given point (e.g., *Find restaurants within 5km of user coordinates*).
* **Under the Hood:** Elasticsearch indexes coordinates using spatial index structures like **BKD trees** (Block-KD trees), enabling rapid multi-dimensional spatial filtering without scanning the entire dataset.

### Aggregations & Analytical Engine
Aggregations form the backbone of Elasticsearch's analytical power, enabling real-time metrics and dashboards (crucial for Kibana or metric analytics).
* **Metric Aggregations:** Calculations computed over document fields (e.g., `avg`, `sum`, `min`, `max`, `percentiles`).
* **Bucket Aggregations:** Grouping documents into buckets or criteria (e.g., grouping sales by country, date histograms by hour, or terms aggregation for top tags).
* **Doc Values:** Aggregations do not rely on inverted indices; instead, they use **Doc Values**—a columnar, disk-based data structure built at index time that maps documents to field values. This keeps memory usage predictable and allows lightning-fast analytical passes.

### Refresh Interval & Index Lifecycle Management (ILM)
* **Refresh Interval:** Controls when ingested data becomes searchable (`index.refresh_interval`, default: 1s). Lowering or disabling this during bulk imports massively accelerates ingestion throughput.
* **ILM:** Automates moving data through phases: *Hot* (high IOPS NVMe for writes/frequent queries) $\rightarrow$ *Warm* $\rightarrow$ *Cold* $\rightarrow$ *Frozen* (searchable snapshots on cheap object storage like S3) $\rightarrow$ *Delete*.

### Routing Keys
By default, Elasticsearch routes documents to shards using a document ID hash. Custom routing allows you to force documents belonging to the same tenant/user into the *exact same shard*, cutting multi-shard scatter-gather overhead.

## Part 3: Pros and Cons of Elasticsearch

### Pros
* **Blazing Fast Read Performance:** Thanks to Lucene segments, inverted indices, and column-oriented doc values, queries over billions of rows execute in milliseconds.
* **Horizontal Scalability:** You can scale out a cluster seamlessly by adding more data or coordinating nodes.
* **Rich Ecosystem:** Integrates smoothly out-of-the-box with Kibana for visualization, Logstash/Beats for ingestion, and modern ML frameworks for vector search.
* **High Availability:** Automatic cluster rebalancing, node failure detection, and replica failover ensure high uptime.

### Cons
* **High Memory Footprint (JVM Heap):** Elasticsearch runs on the JVM. It is notorious for memory pressure issues, Garbage Collection (GC) pauses, and the dreaded `OutOfMemoryError` if heap sizing and shard counts aren’t meticulously managed.
* **Complex Cluster State Management:** Managing split-brain scenarios, cluster state updates across master nodes, and rolling upgrades in production require specialized operational knowledge.
* **Eventual Consistency / Update Overhead:** Because of how Lucene segments are immutable (updates are implemented as a delete + insert), high-frequency updates/deletes cause heavy background segment merging, leading to "write amplification" and CPU spikes.
* **Cost-Prohibitive Storage:** Maintaining real-time indices entirely in local SSD/NVMe storage can become expensive at petabyte scale without aggressive ILM tiering.