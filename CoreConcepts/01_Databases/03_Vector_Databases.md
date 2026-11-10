# Vector Databases & Semantic Search: System Design Guide
> *A comprehensive engineering guide to embeddings, ANN indexing strategies, write architecture, filtering mechanics, and system design interview trade-offs.*

---

## Table of Contents
1. [The Semantic Search Gap](#1-the-semantic-search-gap)
   - *Why Keyword Search and B-Trees Fail*
2. [What is a Vector Embedding?](#2-what-is-a-vector-embedding)
   - *Geometry of Meaning and Similarity Metrics*
3. [The Scale Problem & Exact vs. Approximate Search](#3-the-scale-problem--exact-vs-approximate-search)
   - *$k$-NN Bruteforce Complexity and the Recall Trade-off*
4. [Deep Dive: ANN Indexing Strategies](#4-deep-dive-ann-indexing-strategies)
   - *HNSW, IVF, LSH, and Annoy*
5. [Advanced Search Patterns](#5-advanced-search-patterns)
   - *Metadata Filtering & Hybrid Search*
6. [Write Path & Index Management Architecture](#6-write-path--index-management-architecture)
   - *Hot/Cold Tiering, Deletes, and Rebuild Strategies*
7. [Database Selection & Memory Sizing](#7-database-selection--memory-sizing)
   - *General-Purpose vs. Purpose-Built Vector Stores*
8. [System Design Interview Playbook & Probes](#8-system-design-interview-playbook--probes)
   - *Architectural Patterns and Common Follow-up Questions*

---

## 1. The Semantic Search Gap

### Why Keyword Search and B-Trees Fail

Traditional databases excel at deterministic lookups ($ID = 12345$) or range scans on sorted datasets (orders placed on `2026-01-01`). Relational databases rely on B-Trees and sorted lists to provide $O(\log N)$ point lookups. 

However, **closeness in meaning cannot be sorted ahead of time**.

#### The TrailHead E-Commerce Scenario
* **User Query:** *"something to keep my feet dry on a muddy hike"*
* **Catalog Item:** *"Ridgeline Goretex Trailboot"*
* **Keyword Search Result:** **0 Matches.** 

Even though the product perfectly fits the user's intent, keyword search (lexical matching) fails because there is no exact string intersection between the query terms and the product title. 

Vector databases bridge this gap by transforming intent and content into a shared geometric space where semantic similarity corresponds to spatial proximity.

---

## 2. What is a Vector Embedding?

### Geometry of Meaning

An **embedding** is a dense, fixed-length array of floating-point numbers representing the semantic essence of an entity (e.g., text, images, user profiles).

$$\vec{v} = [x_1, x_2, x_3, \dots, x_d] \in \mathbb{R}^d$$

Where $d$ represents the dimensionality of the vector (typically $d \in [64, 1536]$).

```
   High-Dimensional Space (Conceptual 2D Reduction)
  +-------------------------------------------------+
  |                                                 |
  |   [ Waterproof Hiker ]   [ Trail Boot ]         |
  |            *                  *                 |
  |                                                 |
  |         [ Rain Jacket ]                         |
  |                *                                |
  |                                                 |
  |                                                 |
  |                                  [ Camp Stove ] |
  |                                         *       |
  +-------------------------------------------------+
```

### Distance Metrics

To calculate how "close" two vectors $\vec{u}$ and $\vec{v}$ are in vector space, three primary distance/similarity metrics are used:

| Metric | Formula | Description | Best Use Case |
| :--- | :--- | :--- | :--- |
| **Cosine Similarity** | $\cos(\theta) = \frac{\vec{u} \cdot \vec{v}}{\|\vec{u}\| \|\vec{v}\|}$ | Measures the angle between two vectors, ignoring magnitude. | Text models, domain embeddings normalized to unit length. |
| **Euclidean Distance ($L_2$)** | $d(\vec{u}, \vec{v}) = \sqrt{\sum_{i=1}^d (u_i - v_i)^2}$ | Measures the straight-line distance between two points. | Normalized vectors, spatial clustering. |
| **Dot Product** | $\text{DP}(\vec{u}, \vec{v}) = \vec{u} \cdot \vec{v} = \sum_{i=1}^d u_i v_i$ | Unnormalized cosine similarity. Scaled by vector length. | Fast computation when vectors are pre-normalized to $\|\vec{v}\| = 1$. |

---

## 3. The Scale Problem & Exact vs. Approximate Search

### $k$-Nearest Neighbors ($k$-NN) / Exact Search

Exact search calculates the distance between the query vector $\vec{q}$ and every vector $\vec{v}_i$ in the dataset, using a min-heap to keep track of the top $k$ items.

* **Computational Complexity:** $O(N \cdot d)$ per query.
* **Calculation Load Example:**
  * Catalog Size ($N$): $1,000,000$ products
  * Dimensions ($d$): $1,536$
  * Floating-Point Operations per query: $1,000,000 \times 1,536 \approx 1.536 \times 10^9$ multiply-add operations.

For small datasets ($N \le 10,000$), exact $k$-NN is optimal and accurate. At scale ($N \ge 1,000,000$), linear scaling becomes computationally intractable for real-time web applications.

### Approximate Nearest Neighbor (ANN) Search

ANN trades a marginal amount of accuracy (**Recall**) for orders-of-magnitude reduction in latency.

$$\text{Recall@}k = \frac{|\text{Retrieved Top-}k \cap \text{True Top-}k|}{k}$$

* **Target Recall in Production:** Typically $90\% - 98\%$ (e.g., finding 9 out of the true top 10 items).

---

## 4. Deep Dive: ANN Indexing Strategies

### 1. Hierarchical Navigable Small World (HNSW)

HNSW is a multi-layer graph index inspired by **Skip Lists**.

```
Layer 2 (Sparse Edge Express Lane)   ( Node A ) --------------------------> ( Node Z )
                                         |                                     |
                                         v                                     v
Layer 1 (Medium Density)            ( Node A ) ------> ( Node M ) --------> ( Node Z )
                                         |                |                    |
                                         v                v                    v
Layer 0 (Dense Graph - All Nodes)   ( Node A ) -> (B) -> (M) -> (P) -> (Q) -> (Z)
```

* **Mechanism:**
  * **Layer 0:** Contains all vectors connected in a proximity graph.
  * **Upper Layers:** Contain probabilistic subsets of nodes with long-range links ("express lanes").
  * **Search:** Starts at top-layer entry points, greedily moves toward the target vector, drops to the next layer down upon reaching a local minimum, and repeats until completing local search on Layer 0.
* **Trade-Offs:**
  * **Recall / Latency:** Outstanding ($\ge 95\%$ recall in milliseconds).
  * **Memory Footprint:** Very High. Graph topology must reside in RAM.
  * **Insert Speed:** Slow ($O(\log N)$ graph traversals per insertion to link neighbors across layers).

### 2. Inverted File Index (IVF)

IVF uses **$K$-Means clustering** to divide vector space into partitions centered around centroids.

```
       [ Cluster Centroid 1 ]           [ Cluster Centroid 2 ]
       /          |         \           /          |         \
   (Vec A)     (Vec B)    (Vec C)   (Vec D)     (Vec E)    (Vec F)
```

* **Mechanism:**
  * Vectors are assigned to their nearest cluster centroid.
  * At query time, the system identifies the $n_{\text{probe}}$ nearest centroids and searches only vectors within those clusters, skipping $\sim 99\%$ of the dataset.
* **Key Parameter:** $n_{\text{probe}}$ (Number of clusters to inspect). Higher $n_{\text{probe}}$ increases recall at the cost of latency.
* **Trade-Offs:**
  * **Memory Footprint:** Lower than HNSW (no graph edges).
  * **Build Speed:** Fast initial build; handles inserts cleanly until centroids drift.
  * **Maintenance:** Requires periodic retraining as data distribution changes.

### 3. Locality Sensitive Hashing (LSH)

LSH uses random hyperplanes to partition vector space, turning spatial proximity into hash bucket collisions.

* **Mechanism:** $k$ random lines/planes yield a $k$-bit hash code based on which side of each plane a vector falls. Vectors with matching hash buckets are evaluated. Multiple hash tables are queried in parallel to improve recall.
* **Trade-Offs:** Instant builds and fast lookups, but lower recall-to-latency efficiency than HNSW.

### 4. Annoy (Approximate Nearest Neighbors Oh Yeah)

Developed by Spotify, Annoy uses random projection trees to partition space recursively into binary trees (creating a forest of trees).

* **Trade-Offs:** Index can be mapped directly to disk (`mmap`), allowing static files to be shared instantly across multiple processes. **Immutable index**: dynamic updates require a full rebuild.

---

### Index Strategy Comparison

| Index Type | Indexing Mechanism | Memory Usage | Insert Speed | Search Latency | Recall | Ideal Use Case |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **HNSW** | Multi-layer Graph | **High** (RAM) | Slow | **Very Low** | **Very High** | High-throughput, real-time read workloads. |
| **IVF** | $K$-Means Clustering | Low / Medium | Medium | Low | Medium / High | Large datasets requiring lower memory footprint. |
| **LSH** | Hyperplane Hashing | Low | Fast | Low | Low / Medium | Streaming high-velocity data. |
| **Annoy** | Forest of Trees | Low (`mmap`) | Immutable | Low | Medium | Static read-heavy catalogs updated in nightly batches. |

---

## 5. Advanced Search Patterns

### Filtered Vector Search

Real-world queries combine semantic vectors with transactional metadata (e.g., *"Waterproof boots AND price < $150 AND in_stock = true"*).

```
                      +-----------------------------+
                      | Query: "Waterproof boots"   |
                      | Filters: price < $150       |
                      +--------------+--------------+
                                     |
         +---------------------------+---------------------------+
         |                                                       |
         v                                                       v
[ Post-Filtering ]                                      [ Pre-Filtering ]
1. Run vector search (Top 100).                         1. Filter metadata index (price < $150).
2. Filter results by price < $150.                      2. Run exact/vector search on subset.
Risk: Low result count if filter is restrictive.        Risk: HNSW graph traversal broken if disconnected.
```

1. **Post-Filtering:** Run ANN search for top $K$ items, then filter out non-matching items.
   * *Problem:* Highly selective filters may return fewer than desired results.
2. **Pre-Filtering:** Filter by metadata first, then search within the subset.
   * *Problem:* Carving out arbitrary subsets breaks pre-built HNSW graph connections. (If the subset is very small, fallback to exact bruteforce $k$-NN search).
3. **Single-Pass / Native Filtering:** Integrated indexing (e.g., Pinecone metadata inverted index, Elasticsearch graph-walk filter).

### Hybrid Search

Combines lexical (BM25 keyword match) and vector (dense semantic match) search pipelines using **Reciprocal Rank Fusion (RRF)**:

$$\text{RRF Score}(d \in D) = \sum_{m \in M} \frac{1}{k + r_m(d)}$$

Where $r_m(d)$ is the rank position of document $d$ in search result list $m$.

---

## 6. Write Path & Index Management Architecture

Because updating graphs or clusters dynamically is computationally intensive, vector databases adopt architectural patterns similar to LSM Trees.

```
                           +---------------------------+
                           |       Write Request       |
                           +-------------+-------------+
                                         |
                         +---------------+---------------+
                         |                               |
                         v                               v
            +------------------------+      +------------------------+
            |      Hot Index         |      |  Cold Index (HNSW/IVF) |
            |  (In-Memory / Buffer)  |      |   (Static / Read-Only) |
            +-----------+------------+      +-----------+------------+
                        |                               |
                        +---------------+---------------+
                                        |
                                        v
                            +-----------------------+
                            | Combined Search Merge |
                            +-----------------------+
```

* **Hot/Cold Tiering:** New writes go into a small, unindexed memory buffer (bruteforced during read queries). Periodically, hot data is merged into a static cold index via background rebuilds.
* **Deletes:** Soft deletes are applied by flagging vector IDs in a bitmask. Soft-deleted vectors continue consuming memory until a consolidation/rebuild phase purges them.
* **Blue-Green Rebuild Strategy:** Large indices are rebuilt out-of-band in a shadow system and swapped atomically into production once completed.

---

## 7. Database Selection & Memory Sizing

### RAM Sizing Estimation

Calculating RAM requirements for a $1,500$-dimension float32 vector index:

$$\text{Vector Size} = 1536 \text{ dimensions} \times 4 \text{ bytes (float32)} = 6,144 \text{ bytes} \approx 6 \text{ KB}$$

| Vector Count ($N$) | Raw Vector Memory | Graph Overhead ($\sim 25\%$) | Total Memory Needed | Recommended Architecture |
| :--- | :--- | :--- | :--- | :--- |
| **100,000** | $\sim 600\text{ MB}$ | $\sim 150\text{ MB}$ | **$< 1\text{ GB}$** | In-Process / Single DB |
| **1,000,000** | $\sim 6\text{ GB}$ | $\sim 1.5\text{ GB}$ | **$\sim 8\text{ GB}$** | Single Instance (pgvector/Redis) |
| **10,000,000** | $\sim 60\text{ GB}$ | $\sim 15\text{ GB}$ | **$\sim 80\text{ GB}$** | Large Memory Instance / Sharded |
| **100,000,000** | $\sim 600\text{ GB}$ | $\sim 150\text{ GB}$ | **$\sim 800\text{ GB}$** | Purpose-Built Distributed Vector DB |

### Ecosystem Decision Matrix

| Technology | Category | Pros | Cons | Ideal Use Case |
| :--- | :--- | :--- | :--- | :--- |
| **pgvector (PostgreSQL)** | Extension | Single system, standard joins, ACID compliant. | High-scale RAM limitations. | $< 10\text{M}$ vectors; existing Postgres stack. |
| **Elasticsearch / OpenSearch** | Hybrid Engine | Excellent text + vector hybrid capabilities. | Memory heavy; expensive at scale. | E-commerce hybrid search workloads. |
| **Redis Vector Search** | Cache / In-Memory | Extremely low latency lookups. | Expensive RAM storage. | Real-time recommendations & caching. |
| **Pinecone** | Managed Cloud | Fully managed, auto-scaling, integrated filtering. | Proprietary, vendor lock-in, cost per query. | Operational simplicity preferred over cost control. |
| **Milvus / Qdrant** | Purpose-Built OSS | Distributed, optimized for billions of vectors. | Operational complexity of multi-component cluster. | Large scale ($> 100\text{M}$ vectors), self-hosted requirements. |

---

## 8. System Design Interview Playbook & Probes

### Standard Canonical Architecture Pattern

In system design interviews, treat the vector database as an **index**, not your primary storage layer or source of truth.

```
+--------+       1. Embed Query        +---------------+
| Client | --------------------------->| Embedding Model|
+--------+                             +-------+-------+
    |                                          | 2. Query Vector
    | 4. Fetch Product Details                 v
    |    (IDs: [102, 405, 881])        +---------------+
    | <------------------------------- | Vector Index  |
    |                                  +---------------+
    v
+-----------------------+
| Primary Database (DB) | (PostgreSQL, DynamoDB, etc.)
+-----------------------+
```

### Key Interview Questions & Trade-offs

#### Q1: "The ML team deployed a new embedding model. How do we update our system?"
* **Core Risk:** Vectors generated by Model A live in a different spatial coordinates domain than vectors generated by Model B. Searching a Model B query vector against a Model A index yields incorrect results.
* **Solution:**
  1. Version all stored vectors with their source model (`model_version = v2`).
  2. Create a parallel index using the new embedding model.
  3. Re-embed the product catalog offline into the new index.
  4. Perform a zero-downtime cutover to the new index and update the query embedding service.

#### Q2: "Should we use 1536-dimension embeddings or 384-dimension embeddings?"
* **Trade-off:**
  * Higher dimensions yield higher representation precision, but increase memory footprint and computation latency linearly.
  * Downscaling from 1536 to 384 dimensions reduces memory usage by **$75\%$** with minimal accuracy drop in many applied domains. Run benchmark experiments to validate quality loss versus infrastructure cost savings.

#### Q3: "When should we NOT use a specialized vector database?"
* When dataset sizes are small ($< 100,000$ items) where exact brute force $k$-NN or standard pgvector indices suffice.
* When standard SQL metadata queries or BM25 lexical keyword searches satisfy product retrieval requirements without semantic understanding.