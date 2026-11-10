# FB Post Search — Staff-Level System Design Guide

> Design a keyword search over Facebook posts: users create and like posts, search by keyword, and sort results by **recency** or **like count**, at a scale of **trillions of posts** and **billions of likes per day**.
>
> All numbers are **illustrative** (decimal SI units) and are meant to drive design decisions, not to claim Meta's real figures.

---

## Table of Contents

1. [Functional Requirements](#1-functional-requirements)
2. [Non-functional Requirements](#2-non-functional-requirements)
3. [Capacity Estimation](#3-capacity-estimation)
4. [Core Entities](#4-core-entities)
5. [API / System Interface](#5-api--system-interface)
6. [Data Flow](#6-data-flow)
7. [High-Level Design (4 FRs with talk tracks)](#7-high-level-design)
8. [Deep Dives (4 NFRs with talk tracks)](#8-deep-dives)
9. [The 8 Interview Questions — Answered with Options, Pros & Cons](#9-the-8-interview-questions)
10. [CAP / PACELC per Component](#10-cap--pacelc-per-component)
11. [Failure Modes & Mitigations](#11-failure-modes--mitigations)
12. [Scalability & Sharding Summary](#12-scalability--sharding-summary)
13. [Senior vs. Staff Differentiators](#13-senior-vs-staff-differentiators)
14. [45-Minute Interview Time Allocation](#14-45-minute-interview-time-allocation)
15. [Quick-Reference Cheatsheet](#15-quick-reference-cheatsheet)
16. [Related Guides](#16-related-guides)

---

## 1. Functional Requirements

**In scope (core)**

| # | Requirement |
|---|---|
| FR1 | Users can **create posts** (text). |
| FR2 | Users can **like** posts (and unlike). |
| FR3 | Users can **search posts by keyword** (one or more keywords). |
| FR4 | Users can **sort search results by recency or like count**, with pagination. |

**Explicitly out of scope** (say this out loud — it bounds the problem)

- Fuzzy matching / typo tolerance, stemming beyond basic normalization, semantic/vector search.
- Personalized ranking (friends-first, ML relevance). Sort is purely recency or likes.
- Privacy/visibility filtering (friends-only posts). *Staff note: call out that adding it later breaks query-result caching — see §8.1.*
- Images, video, hashtags-as-entities, comments search.

---

## 2. Non-functional Requirements

| # | NFR | Target | Why it matters |
|---|---|---|---|
| NFR1 | **Low search latency** | p99 < 500 ms end-to-end | Search is interactive. |
| NFR2 | **Massive scale** | Trillions of posts indexed; ~1B searches/day | Index can't live on one machine. |
| NFR3 | **Freshness** | New post searchable within **~1 minute**; like counts reflected within ~minutes | Search must feel live (breaking news), but strict real-time is not required. |
| NFR4 | **High write throughput** | ~35k posts/s and ~350k likes/s at peak | Likes are the dominant write load. |
| NFR5 | **High availability** | 99.99% for search; prefer stale over unavailable | AP system with eventual consistency. |
| NFR6 | **Storage / cost efficiency** | Index cost must stay sub-linear vs. raw corpus growth | Petabyte-scale index. |

**Not required:** 100% recall (missing a few results is acceptable), strict read-your-writes on search.

> **Anchor constraint:** the like-count sort order is the hard part. Recency is append-only; likes mutate the ordering of *old* posts constantly. Almost every non-obvious decision flows from that.

---

## 3. Capacity Estimation

**Assumptions (illustrative):** 1B DAU; 1 post/user/day; 10 likes/user/day; 1 search/user/day; peak = 3× average; 86,400 s/day ≈ 10⁵.

### Throughput

| Workload | Daily | Avg/s | Peak/s |
|---|---|---|---|
| Post creation | 1B | ~11.6k | **~35k** |
| Likes | 10B | ~116k | **~350k** |
| Searches | 1B | ~11.6k | **~35k** |

→ Writes outnumber searches ~10:1, and **likes are ~90% of writes**.

### Storage

| Item | Math | Result |
|---|---|---|
| Total posts (10 yrs) | 1B × 365 × 10 | **~3.65 T posts** |
| Raw post data | 3.65T × 1 KB | ~3.65 PB (×3 replication ≈ 11 PB) |
| Distinct terms per post | ~100 words → ~40 unique non-stopwords | 40 |
| Postings (one sort order) | 3.65T × 40 | **~146 T postings** |
| Index — naive in-memory (Redis ZSET ~50–80 B/entry, 2 sort orders) | 146T × ~64 B × 2 | **~19 PB RAM** ❌ |
| Index — compressed on disk (delta + varint ≈ 4 B/entry, 2 orders) | 146T × 4 B × 2 | **~1.2 PB disk** ✅ |
| Hot tier (last ~30 days in RAM) | 30B posts × 40 × 2 × ~64 B | **~150 TB RAM** → ~300 nodes @ 512 GB (×3 replicas ≈ 900) |

**Conclusion:** the full index can't be in RAM. We need a **hot (RAM) / cold (compressed SSD/blob) tiered index**. This number is what justifies the tiering design — say it explicitly.

### Index write fan-out

| Source | Math | Index ops/s (peak) |
|---|---|---|
| New posts | 35k × 40 terms × 2 orders | ~2.8M |
| Likes (naive: update every term's likes-index on every like) | 350k × 40 | **~14M** ❌ |
| Likes (milestone/log-scale updates + batching) | ≪ 1% of likes cross a milestone | ~tens of k ✅ |

### Read side

- 35k QPS peak; Zipf-distributed queries → with an ~80% result-cache hit rate, ~7k QPS reach the index.
- Response ≈ 25 results × ~1 KB ≈ 25 KB → ~0.9 GB/s egress at peak (fine).

---

## 4. Core Entities

| Entity | Key fields | Store |
|---|---|---|
| **User** | userId, name | User service (not designed here) |
| **Post** | postId (Snowflake, time-ordered 64-bit), authorId, content, createdAt, **terms[]** (tokenized), status (ACTIVE/DELETED) | Post DB (Cassandra/DynamoDB), partition key = postId |
| **Like** | (postId, userId) composite PK, createdAt | Like DB (Cassandra/DynamoDB) — PK makes likes **idempotent** |
| **LikeCount** | postId → count | Redis counters (fast) + periodically persisted |
| **Inverted Index entry** | term → postings list of (postId, score), two lists per term: `recency`, `likes` | Hot: Redis sorted sets / custom in-memory; Cold: compressed segment files on blob/SSD |
| **QueryCache entry** | normalized(query, sort, cursor) → result page | Redis / CDN |

> Why store `terms[]` on the Post? When a like arrives we must know which index lists to update, and on deletion/edit we must know which postings to remove — without re-tokenizing.

---

## 5. API / System Interface

```http
POST /v1/posts
Body: { "content": "Saw Taylor Swift live tonight!" }
→ 201 { "postId": "1798223344556677" }

PUT /v1/posts/{postId}/likes          # idempotent; userId from auth token
→ 204
DELETE /v1/posts/{postId}/likes       # unlike
→ 204

GET /v1/posts/search?q=taylor+swift&sort=RECENCY|LIKES&limit=25&cursor=<opaque>
→ 200 {
    "results": [ { "postId", "authorId", "snippet", "createdAt", "likeCount" } ],
    "nextCursor": "<opaque>"
  }
```

**Design choices to call out**

- `PUT` for likes (not `POST`) → idempotent; client retries don't double-count.
- `userId` always from the auth token, never from the body.
- **Cursor pagination, not offset.** Cursor encodes `(lastScore, lastPostId)`, so page N is O(log n + limit) instead of skipping N×25 entries, and it's stable while new posts arrive.
- Query semantics: space-separated keywords = **AND** by default.

---

## 6. Data Flow

**Write path (post):**
1. Client → API Gateway → Post Service.
2. Post Service generates Snowflake postId, tokenizes content (lowercase, strip punctuation, remove stopwords) → `terms[]`.
3. Writes Post row to Post DB (durable source of truth) → returns 201.
4. CDC / outbox publishes `PostCreated` to Kafka (partition by postId).
5. Ingestion workers consume → write `(postId, createdAt)` into each term's **recency** list and `(postId, 0)` into each term's **likes** list on the hot index shards.

**Write path (like):**
1. Client → Like Service → idempotent upsert of `(postId, userId)` in Like DB.
2. If the row was new: `INCR likecount:{postId}` and publish `LikeEvent` to Kafka.
3. Like aggregator (Flink / Kafka Streams) windows events per postId (e.g., 10–30 s) → emits net deltas.
4. Index updater updates the **likes** list for each of the post's terms **only when the count crosses a milestone** (1, 2, 4, 8, … or similar).

**Read path (search):**
1. Client → Search Service: normalize query (same tokenizer as ingestion), build cache key.
2. Query cache hit → return.
3. Miss → Index Query Router finds shards for each term → fetch top candidates (single term) or intersect (multi-term).
4. For `sort=LIKES`: over-fetch candidates (e.g., 4×limit), refresh true counts from LikeCount, re-rank.
5. Hydrate postIds → post bodies via Post Cache/Post DB; **drop DELETED posts** (read-time filter).
6. Write page to query cache (short TTL) → return with cursor.

---

## 7. High-Level Design

### Overall architecture

```
                               ┌────────────────┐
           Clients ───────────▶│  API Gateway   │  (auth, rate limit, routing)
                               └───┬────┬────┬──┘
                    ┌──────────────┘    │    └──────────────────┐
                    ▼                   ▼                       ▼
           ┌──────────────┐    ┌──────────────┐        ┌────────────────┐
           │ Post Service │    │ Like Service │        │ Search Service │
           └──────┬───────┘    └──┬────────┬──┘        └──┬──────────┬──┘
                  │               │        │              │          │
                  ▼               ▼        ▼              ▼          ▼
           ┌──────────┐   ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌───────────┐
           │ Post DB  │   │ Like DB  │ │LikeCount │ │  Query   │ │Post Cache │
           │(Cassandra│   │(PK=post, │ │ (Redis)  │ │  Cache   │ │ (Redis)   │
           │ /Dynamo) │   │  user)   │ └────▲─────┘ │(Redis/CDN│ └─────┬─────┘
           └────┬─────┘   └────┬─────┘      │       └──────────┘       │
                │ CDC/outbox   │ events     │                          │
                ▼              ▼            │                          │
           ┌─────────────────────────────┐  │      ┌────────────────────┐
           │            Kafka            │  │      │ Index Query Router │
           │ topics: post-created, likes │  │      └─────────┬──────────┘
           └───────┬──────────────┬──────┘  │                │
                   ▼              ▼         │                ▼
           ┌──────────────┐ ┌──────────────┐│    ┌──────────────────────────┐
           │  Ingestion   │ │    Like      ├┘    │   Inverted Index (HOT)   │
           │   Workers    │ │  Aggregator  │     │ sharded by hash(term)    │
           │ (tokenize,   │ │ (Flink, 30s  │     │ per term: recency ZSET,  │
           │  index)      │ │  windows,    │     │           likes ZSET     │
           └──────┬───────┘ │  milestones) │     └────────────┬─────────────┘
                  │         └──────┬───────┘                  │ age-out / cap
                  └────────┬───────┘                          ▼
                           ▼                     ┌──────────────────────────┐
                  (writes to HOT index)          │ Inverted Index (COLD)    │
                                                 │ compressed segments on   │
                                                 │ SSD / blob (S3)          │
                                                 └──────────────────────────┘
```

**Per-component "why"**

| Component | Why it exists | Alternatives considered |
|---|---|---|
| API Gateway | Auth, rate limiting (search abuse/scrapers), routing | — |
| Post Service + Post DB | Source of truth; Cassandra/Dynamo for write-heavy, key-value access by postId | Postgres sharded — fine, but we never need joins/transactions here |
| Like DB with PK (postId,userId) | Idempotency for free; enables unlike and "did I like this?" | Counter-only — loses dedupe |
| LikeCount (Redis) | O(1) reads of current count for re-ranking & display | Count from Like DB — too slow |
| Kafka | Decouples write latency from indexing; absorbs spikes; replayable for re-index | Direct synchronous indexing — couples availability, no replay |
| Ingestion Workers | Stateless, horizontally scalable tokenization + index writes | Index inside Post Service — couples scaling |
| Like Aggregator | Collapses 350k likes/s into a small number of index updates | Per-like index writes — 14M ops/s |
| Hot Inverted Index | Sub-ms access to recent/top postings | Elasticsearch — see §9 Q2 |
| Cold Index | Cheap storage for the long tail of old postings | Everything in RAM — ~19 PB |
| Query Cache | Zipfian queries → most traffic hits a few thousand queries | — |
| Post Cache | Hydration of 25 posts per page without hitting Post DB | — |

---

### FR1 — Create a post

```
Client ─▶ Gateway ─▶ Post Service ──(1) write──▶ Post DB
                          │                        │ (2) CDC/outbox
                          ◀── 201 {postId}          ▼
                                                 Kafka "post-created"
                                                    │ (3)
                                                    ▼
                                         Ingestion Workers ──(4)──▶ Hot Index shards
                                                                   ZADD recency:{term} createdAt postId
                                                                   ZADD likes:{term}   0         postId
```

**Talk track (~100 s)**

- "Post creation is a write-then-index pipeline; the user only waits for the durable write."
- Post Service generates a **Snowflake postId** — time-ordered, globally unique, no coordination.
- It tokenizes once: lowercase, strip punctuation, drop stopwords, dedupe → `terms[]`, stored on the Post row so later likes and deletes know which index lists to touch.
- Write the Post row to **Cassandra/DynamoDB**, partitioned by postId — even distribution, no hot partitions.
- Return **201 immediately**; indexing is asynchronous — that's what our 1-minute freshness budget buys us.
- A **transactional outbox or CDC** emits `PostCreated` to Kafka — this avoids the dual-write problem where the DB commits but the Kafka publish is lost.
- Kafka is partitioned by postId; **stateless ingestion workers** consume and, for each term, write into the term's recency list (score = createdAt) and likes list (score = 0).
- Writes are idempotent (`ZADD` of the same member is a no-op), so at-least-once delivery is safe.
- If the index is down, Kafka buffers; we catch up on recovery and can **replay to rebuild** the index entirely.
- Trade-off: synchronous indexing would give read-your-writes, but couples post availability to index availability — not worth it.

---

### FR2 — Like a post

```
Client ─▶ Like Service ─(1) upsert IF NOT EXISTS (postId,userId)─▶ Like DB
               │ (2) if new: INCR likecount:{postId} ─▶ LikeCount (Redis)
               │ (3) publish LikeEvent ─▶ Kafka "likes" (partition by postId)
               ▼
        Like Aggregator (Flink, 30 s tumbling window per postId)
               │ (4) newCount crosses milestone (2^k)?
               ▼  yes
        Index Updater: for term in post.terms: ZADD likes:{term} newCount postId
```

**Talk track (~100 s)**

- "Likes are 90% of our write traffic — 350k per second at peak — so the like path is designed to *not* touch the search index on most likes."
- The Like DB uses a composite primary key **(postId, userId)**: a duplicate like is a no-op upsert, so `PUT /likes` is naturally idempotent and retries can't inflate counts.
- Only if the row is new do we `INCR` the post's counter in **Redis** — that's the authoritative-enough count used for display and final re-ranking.
- We publish a LikeEvent to Kafka **partitioned by postId**, so all likes for a post land on one aggregator partition.
- The **Flink aggregator** sums deltas over a 10–30 second window — a viral post with 50k likes in that window produces **one** event, not 50k.
- Second reduction: we update the term-level likes index **only when the count crosses a milestone** — powers of two. Getting to 1M likes takes ~20 index updates instead of a million.
- The index score is therefore approximate, within 2×; we fix exact order at query time by re-ranking candidates with the true count from Redis.
- Net effect: naive 14M index ops/s drops to tens of thousands.
- Unlike is a delete plus `DECR`; we don't decrement the index on milestones — slight over-ranking is acceptable and self-corrects via re-rank.

---

### FR3 — Search by keyword

```
Client ─▶ Search Service ─(1) normalize & tokenize "Taylor Swift!" → [taylor, swift]
              │ (2) GET qc:{taylor swift|RECENCY|cursor} ─▶ Query Cache ── hit ─▶ return
              │ miss
              ▼
        Index Query Router ─(3) hash(term) → shard ─▶ Hot Index shards
              │ (4) single term: ZREVRANGEBYSCORE recency:{taylor} (cursor) LIMIT 25
              │     multi-term: bigram key "taylor_swift" if exists, else intersect
              ▼
        Hydrate ─(5) MGET post:{id...} ─▶ Post Cache ─miss─▶ Post DB
              │ (6) drop status=DELETED
              ▼
        (7) SET qc:... TTL 30–60 s ─▶ return results + nextCursor
```

**Talk track (~100 s)**

- "Start simple: the naive version is `SELECT … WHERE content LIKE '%taylor%'` — a full table scan over trillions of rows. It's correct and useless, which motivates an **inverted index**."
- An inverted index maps each term to its postings list — the postIds containing that term — so lookup cost depends on the result size, not the corpus size.
- The Search Service normalizes the query with **exactly the same tokenizer** as ingestion; a mismatch is a classic silent-miss bug.
- First stop is the **query cache**: queries are Zipfian, so a small set of hot queries serves most traffic.
- On a miss, the router hashes each term to its index shard and reads the top postings.
- Single keyword: one range read on one sorted set. Multiple keywords: use a precomputed **bigram list** if one exists, otherwise intersect — detailed in §9 Q5.
- We get back only postIds; we **hydrate** the 25 post bodies with a batched `MGET` from the post cache.
- During hydration we **filter deleted posts** — the index is eventually consistent, but deletion visibility must not be.
- Cache the page for 30–60 s — within our freshness budget — and return with an opaque cursor.

---

### FR4 — Sort by recency and like count

```
Per term, two postings lists:

 recency:{taylor}  (score = createdAt)          likes:{taylor}  (score = milestone likeCount)
 ┌──────────┬─────────────┐                      ┌──────────┬───────┐
 │ postId   │ createdAt   │                      │ postId   │ score │
 │ 9001     │ 1762500000  │ ◀ newest             │ 4123     │ 2^20  │ ◀ most liked
 │ 8800     │ 1762499990  │                      │ 7777     │ 2^19  │
 │  …       │  …          │                      │  …       │  …    │
 └──────────┴─────────────┘                      └──────────┴───────┘

sort=LIKES:  fetch top 4×limit from likes:{term}
             ─▶ MGET likecount:{id…} (true counts) ─▶ re-sort ─▶ take 25
```

**Talk track (~100 s)**

- "We have two orderings with very different mutation patterns, so we keep **two postings lists per term** rather than sorting at query time."
- Sorting at query time — fetch every 'taylor' posting and sort — is O(n log n) over potentially billions of entries; it can't meet 500 ms.
- **Recency list:** score = createdAt. It's effectively append-only — new posts always go on the head — so it's cheap to maintain and trivially paginated with a `(createdAt, postId)` cursor.
- Trap worth naming: don't use the 64-bit Snowflake ID as a Redis score — scores are doubles with 53 bits of exact precision, so IDs collide. Use seconds-precision createdAt as score, postId as the tie-breaker.
- **Likes list:** score = like count, but updated only at milestones, so it's approximate within 2×.
- At query time we **over-fetch** — say 4× the page size — pull true counts from Redis in one `MGET`, re-sort, and return the top 25. Over-fetch covers the 2× approximation band.
- Cost of two lists: ~2× index storage. Alternative is one list plus query-time sorting — cheaper storage, unacceptable latency. We pay storage to buy latency, then claw storage back with tiering and caps (§9 Q8).
- Pagination is by cursor on both lists; for likes we accept that order can shift slightly between pages — a known, acceptable UX trade-off.

---

## 8. Deep Dives

### 8.1 NFR1 — Search latency p99 < 500 ms

```
 Request ─▶ CDN/edge cache ─▶ Query Cache (Redis) ─▶ Hot index (RAM, top-K) ─▶ Cold index (rare)
             ~10 ms              ~2 ms                 ~5–20 ms                  ~50–150 ms
```

**Talk track (~100 s)**

- "Latency comes from four layers, and the goal is that the vast majority of requests never reach the slow ones."
- **Layer 1 — result caching.** Search queries are Zipfian. A Redis query cache keyed by normalized `(query, sort, cursor)` with a 30–60 s TTL absorbs most traffic. Because results aren't personalized in our scope, we can even cache page 1 at the **CDN edge**.
- **Staff caveat:** the moment privacy filtering is added, results become per-viewer and edge caching breaks — we'd cache the un-filtered candidate set and filter per user after.
- **Layer 2 — cache stampede protection.** When "taylor swift" expires during a concert, thousands of misses hit at once; use **request coalescing (single-flight)** so one request rebuilds and others wait, plus jittered TTLs.
- **Layer 3 — precomputed, pre-sorted lists.** We never sort at query time; reading the top 25 from a sorted set is O(log n + 25).
- **Layer 4 — bound the work.** Cursor pagination, over-fetch at most 4×, and a hard cap on intersection candidates so a pathological query can't run unbounded.
- Hydration uses a single batched `MGET` against the post cache — one round trip, not 25.
- Multi-term queries hit 2–3 shards in **parallel**, with per-shard timeouts and **hedged requests** to replicas to cut tail latency.
- If the cold tier is slow, we return hot-tier results and mark the response partial — we prefer fast-and-slightly-incomplete, consistent with our recall NFR.

---

### 8.2 NFR2 — Scale to trillions of posts

```
                      Term-partitioned (chosen)                Document-partitioned
 hash("taylor") ─▶ Shard 17 (all 'taylor' postings)     each shard indexes a subset of posts
 hash("swift")  ─▶ Shard 42                             query ─▶ scatter to ALL shards ─▶ gather
 + time tiering: HOT (≤30 days, RAM) │ COLD (compressed segments, SSD/S3)
 + hot-term splitting: "taylor" ─▶ taylor#0..taylor#7 sub-shards
```

**Talk track (~100 s)**

- "Our math says the naive index is ~19 PB of RAM, so we need two scaling axes: **sharding** and **tiering**."
- **Sharding choice is the key Staff decision, because it determines which query becomes an expensive fan-out.**
- **Document-partitioned:** each shard indexes its own subset of posts. Writes are local — a post touches one shard — but *every* query scatters to *all* shards, so tail latency is governed by the slowest of hundreds of shards.
- **Term-partitioned:** shard by hash(term). A single-keyword query hits exactly one shard; a two-keyword query hits two. Writes fan out to ~40 shards per post, but writes are async and spread evenly.
- For a read-latency-critical system, **term-partitioning wins**; we pay with write fan-out, which Kafka absorbs.
- Term-partitioning's weakness is **hot terms** — "taylor" during a tour. We split hot terms into N sub-shards (`taylor#0..7` by postId hash) and add read replicas; the router merges.
- **Tiering by time:** posts from the last ~30 days, plus each term's top-K by likes, live in RAM — that's ~150 TB, a few hundred nodes. Everything older is compressed into immutable segment files on SSD/S3.
- Use **consistent hashing** with virtual nodes so adding capacity moves only ~1/N of terms.
- Kafka replay plus Post DB as source of truth means any shard can be **rebuilt** from scratch.

---

### 8.3 NFR3 + NFR4 — Freshness (<1 min) under high write throughput

```
 Posts 35k/s ─┐                      ┌─▶ Ingestion workers (autoscale on lag) ─▶ Hot index
              ├─▶ Kafka (partitioned)┤
 Likes 350k/s ┘                      └─▶ Flink: window 30 s/postId ─▶ milestone filter ─▶ Index updater
                                                    350k/s ─────────▶ ~tens of k/s
```

**Talk track (~100 s)**

- "Freshness and write throughput pull against each other, so we make the budget explicit: one minute from post to searchable."
- Every write goes to the source-of-truth DB first, then to **Kafka**, which decouples the user-facing write latency from indexing and absorbs bursts like New Year's midnight.
- **Posts** are cheap to index — 35k/s × 40 terms ≈ 2.8M append operations/s, spread across hundreds of term shards.
- **Likes** are the real problem. Two reductions: **windowed aggregation** in Flink collapses a burst of likes on one post into a single delta per 30 s; then **milestone updates** only touch the index when the count crosses a power of two.
- Together those turn ~14M index writes/s into tens of thousands — over two orders of magnitude.
- Ingestion workers are stateless and **autoscale on consumer lag**; consumer lag in seconds *is* our freshness SLI, and we alert when it approaches 45 s.
- **At-least-once** delivery is safe because index writes are idempotent — `ZADD` with the same member — and milestone writes are set-to-value, not increment.
- Backpressure: if the index falls behind, Kafka retains the backlog; the only user-visible effect is reduced freshness, not lost data.
- Viral posts are a hot key in Kafka; aggregating per postId in Flink state means one partition handles it without ever writing per-like.

---

### 8.4 NFR5 — High availability (99.99%), stale over unavailable

**Talk track (~100 s)**

- "Search is an **AP** system: we'd rather serve results that are 60 seconds stale than return an error."
- Every hot-index shard has **3 replicas across availability zones**; reads go to any healthy replica, and the router uses health checks plus hedged requests.
- Index writes are asynchronous, so the write path stays up even when the index is degraded — Kafka buffers.
- If a whole term shard is unavailable, we **degrade gracefully**: serve from the query cache even past TTL, or from the cold tier, and flag partial results.
- The index is a **derived, rebuildable** structure. Post DB plus Kafka retention lets us rebuild any shard, so we don't need expensive synchronous replication on the index itself.
- Post DB and Like DB use Cassandra/DynamoDB with quorum writes — tunable replication with `W+R>N`, multi-AZ.
- There is exactly one **CP-flavored exception**: deletions and takedowns. A deleted post must never be served even if the index is stale, so we check status at **hydration** against the authoritative post record and asynchronously purge postings.
- Multi-region: each region has its own index replica fed by a cross-region Kafka mirror; search is served regionally, and we accept slight cross-region freshness lag.
- Blast-radius control: rate limits on search, circuit breakers between Search Service and index shards, and load shedding of `sort=LIKES` deep pages first under overload.

---

## 9. The 8 Interview Questions

### Q1. How will users create posts and like them?

- **Posts:** `POST /posts` → Post Service → Snowflake ID → tokenize → write to Post DB (Cassandra/Dynamo, PK postId) → 201 → outbox/CDC → Kafka → ingestion workers update index. (See FR1.)
- **Likes:** `PUT /posts/{id}/likes` → idempotent upsert into Like DB with PK (postId, userId) → if new, `INCR` Redis count → Kafka → aggregator → milestone index update. (See FR2.)

| Option for like counting | Pros | Cons |
|---|---|---|
| `UPDATE posts SET likes = likes + 1` in Post DB | Simple | Hot-row contention on viral posts; no dedupe |
| Like table + `COUNT(*)` on read | Exact, deduped | O(likes) per read — way too slow |
| **Like table (dedupe) + Redis counter (chosen)** | Idempotent, O(1) reads | Counter can drift → periodic reconcile from Like table |
| Sharded counters (N sub-counters per post) | Handles extreme hot posts | Read must sum N keys; use only for hottest posts |

### Q2. How will users search by keyword? (start simple)

| Option | Pros | Cons |
|---|---|---|
| SQL `LIKE '%kw%'` | Trivial | Full scan over trillions of rows — unusable |
| DB full-text index (Postgres GIN) | No new infra | Doesn't scale past one large cluster; sort-by-likes still hard |
| **Elasticsearch / OpenSearch** | Production-grade inverted index, analyzers, sharding built in | Every like = document reindex (delete + insert into segments) → merge pressure at 350k/s; enormous cluster cost at trillions of docs; less control over tiering |
| **Custom inverted index (chosen for the interview)** | Full control: two sort orders, milestone updates, hot/cold tiers | You own build, ops, and rebuild tooling |

- Simple version: Redis sorted sets — `recency:{term}` and `likes:{term}`; search = `ZREVRANGE` on one key.
- Staff framing: "Elasticsearch is the right answer for most companies. At this scale, the like-driven mutation rate and the cost of indexing trillions of documents is exactly why Meta built a custom index (Unicorn-style). I'll design the custom one so we can discuss the trade-offs."

### Q3. How will users sort by recency and like count?

| Option | Pros | Cons |
|---|---|---|
| Fetch all matches, sort at query time | One index, minimal storage | O(n log n) over millions/billions — fails latency NFR |
| **Two pre-sorted postings lists per term (chosen)** | O(log n + k) reads for either sort | ~2× index storage; likes list needs continuous updates |
| One list + secondary like-count lookup | Less storage | Still must scan all matches to find top-liked |

- Recency score = createdAt (seconds) with postId tie-break; likes score = milestone count, then **over-fetch and re-rank** with true counts. (See FR4.)

### Q4. How can we scale keyword search to trillions of posts?

1. **Term-partition** the index by `hash(term)` with consistent hashing (single-term query → 1 shard). Document-partitioning rejected because every query becomes scatter-gather.
2. **Split hot terms** into sub-shards + read replicas.
3. **Tier by time and popularity:** hot RAM tier (last ~30 days + top-K by likes per term), cold tier of immutable compressed segments on SSD/S3.
4. **Cap** postings returned/stored in hot tier per term (users rarely page past ~10 pages).
5. **Cache** results at Redis and CDN.

| Sharding option | Single-term query | Multi-term query | Writes | Hot-spot risk |
|---|---|---|---|---|
| **By term (chosen)** | 1 shard | k shards + cross-shard intersect | ~40 shards/post | Hot terms → split |
| By document | All shards | All shards (local intersect) | 1 shard/post | Low; tail latency high |
| By time | All time shards (newest first, early stop) | Same | Newest shard only | Newest shard is hot |
| Hybrid term × time | 1 shard per tier | k per tier | Recent tier only | Moderate |

### Q5. How will the system handle multi-keyword queries ("taylor AND swift")?

| Option | Pros | Cons |
|---|---|---|
| **Intersect postings at query time** — start from smallest list, probe the larger | No extra storage; works for any combination | Cross-shard (term-partitioned); large lists ("taylor" + "swift") are expensive |
| Merge-join on both recency lists (both sorted by createdAt), stop at 25 matches | Early termination; streaming | Only works when both lists share the sort key; hard for approximate likes scores |
| **Bigram/phrase index** — index "taylor_swift" as its own term at ingestion | Single-shard, single read — as fast as one keyword | Storage: up to n-1 bigrams/post (~doubles postings) |
| Document-partitioning | Intersection is shard-local | Every query fans out to all shards |

**Recommended hybrid**
- Build bigram lists **selectively** — only for bigrams that are frequent in **query logs** (e.g., top ~1M query bigrams), refreshed daily. Avoids the storage explosion while making popular phrase queries a single read.
- Otherwise intersect: fetch the smaller list first; if it's small, probe the larger list by membership (`ZSCORE`); if both are huge, merge-join the recency lists with early termination; cap candidates and return partial results if the cap is hit.
- Semantics: bigram "taylor_swift" means *adjacent*; AND means *anywhere*. Note the subtle difference — bigram results are a subset, so for strict AND you union bigram results with intersection results, or accept phrase semantics as a product decision.

### Q6. How do we keep queries fast with many results ("taylor swift")?

- **Never materialize all results.** We only need the top 25 per page — read the head of a pre-sorted list.
- **Query result cache** (Redis, TTL 30–60 s) + **CDN** for page 1; Zipf means a few hot queries dominate.
- **Single-flight / request coalescing** + jittered TTL to prevent stampedes when a hot query expires.
- **Precomputed bigram lists** for hot phrases (Q5) → one shard, one read.
- **Cursor pagination** — O(log n) seek, not offset skipping.
- **Bounded over-fetch** (4×) for likes re-rank.
- **Hot-term sub-sharding + replicas** to spread read load for the hottest terms.
- Optional: **proactively warm** cache for trending queries detected from the query stream.

### Q7. How do we make ingestion scalable and fast with millions of posts and billions of likes?

- **Kafka** between source-of-truth writes and indexing: buffering, replay, independent scaling.
- **Outbox/CDC** to avoid dual-write loss.
- **Stateless ingestion workers** autoscaled on consumer lag (lag = freshness SLI).
- **Like aggregation** (Flink, per-postId windows) → one delta per post per window.
- **Milestone (log-scale) updates** to the likes index → O(log likes) index writes per post.
- **Idempotent writes** (ZADD set-semantics, set-to-value milestones) → at-least-once is safe.
- **Batch writes** to index shards (pipeline/MULTI per shard) to amortize network round trips.
- **Hot-post handling:** per-postId Kafka partitioning + in-memory aggregation; sharded counters only for the hottest posts.

| Like-to-index option | Index writes/s (peak) | Accuracy | Notes |
|---|---|---|---|
| Update every term on every like | ~14M | Exact | Infeasible |
| Windowed batching only | ~1–2M | Exact at window end | Still heavy |
| **Batching + milestones (chosen)** | ~tens of k | Within 2×, fixed by re-rank | Best trade-off |
| Periodic offline rebuild of likes lists (hourly) | Batch | Stale up to 1 h | Good for cold tier |

### Q8. How can we optimize storage?

| Technique | Saving | Trade-off |
|---|---|---|
| **Hot/cold tiering** (RAM only for last ~30 days + top-K per term) | ~19 PB RAM → ~150 TB RAM + ~1 PB disk | Old/deep results slower (~100 ms) |
| **Cap hot lists** (e.g., top 1k by likes, recent N per term) | Bounded RAM per term | Very deep pagination goes to cold tier |
| **Postings compression** — sort postIds, delta-encode, varint / PForDelta / Roaring bitmaps | ~8–16 B → ~2–4 B per posting | CPU to decode; immutable segments only |
| **Store only postIds in index**, not content | Index is ids + scores only | Requires hydration step |
| **Stopword removal + normalization** | Removes the largest postings lists ("the", "a") | Can't search stopwords alone |
| **Selective bigrams** (query-log driven) instead of all bigrams | Avoids ~2× blow-up | Rare phrases fall back to intersection |
| **Drop/expire** postings for rare one-off terms in very old posts, or cold-only | Long-tail shrink | Lower recall on ancient rare terms |
| Erasure coding for cold segments on blob storage | ~1.5× vs. 3× replication | Slower recovery |

---

## 10. CAP / PACELC per Component

| Component | CAP | PACELC | Rationale |
|---|---|---|---|
| Post DB | AP | PA/EL | High-availability writes; quorum tunable |
| Like DB | AP | PA/EL | Idempotent PK upsert — no LWT needed |
| LikeCount (Redis) | AP | PA/EL | Approximate counts acceptable; reconciled periodically |
| Inverted Index (hot/cold) | AP | PA/EL | Derived & rebuildable; ~1 min staleness OK |
| Query Cache | AP | PA/EL | TTL-bounded staleness |
| **Deletion / takedown visibility** | **CP-flavored** | PC/EC | Must never serve deleted/illegal content → authoritative check at hydration |

> System-wide: AP/EL with **one named CP exception** (deletion visibility).

---

## 11. Failure Modes & Mitigations

| Failure | Impact | Mitigation |
|---|---|---|
| Index shard down | Missing results for its terms | 3 replicas/AZ; serve stale cache; partial-result flag |
| Kafka consumer lag spike | Freshness SLO breached | Autoscale on lag; alert at 45 s; prioritize post topic over likes |
| Hot term ("taylor" during tour) | Shard CPU saturation | Sub-shard hot terms; read replicas; query cache; trending warm-up |
| Cache stampede on viral query | Thundering herd to index | Single-flight, jittered TTL, serve-stale-while-revalidate |
| Viral post like storm | Hot Kafka partition, counter contention | Per-postId aggregation; sharded counters for top posts |
| Dual-write loss (DB ok, Kafka publish fails) | Post never indexed | Transactional outbox / CDC |
| Tokenizer mismatch (query vs. ingest) | Silent misses | Shared tokenizer library, versioned; reindex on change |
| Deleted post still in index | Policy/legal violation | Hydration-time status filter + async posting purge |
| Like counter drift | Wrong ordering/display | Periodic reconciliation from Like DB |
| Poison message in Kafka | Consumer stuck | DLQ after N retries; skip & alert |
| Index corruption / bad deploy | Wrong results at scale | Rebuild from Post DB + Kafka replay; blue/green index versions |
| Snowflake ID used as double score | Collisions / misordering | Score = createdAt seconds; postId tie-break |

---

## 12. Scalability & Sharding Summary

```
                     ┌──────────────── Index Query Router ────────────────┐
                     │ consistent hash(term) → shard; hot-term map        │
                     └──────┬─────────────────┬───────────────────┬───────┘
                            ▼                 ▼                   ▼
                     Shard A (r=3)     Shard B (r=3)       "taylor" → taylor#0..#7
                     HOT: ≤30d, top-K  HOT …               (hot-term sub-shards)
                            │ age-out / cap overflow
                            ▼
                     COLD segments (immutable, delta+varint compressed, SSD/S3)
```

- **Post DB:** partition by postId (uniform).
- **Like DB:** partition by postId (all likes for a post co-located → reconcile cheaply).
- **Kafka:** posts partitioned by postId; likes by postId (aggregation locality).
- **Index:** term-partitioned, consistent hashing, hot-term splitting, time tiering.
- **Search Service / ingestion workers:** stateless, horizontally scaled.

---

## 13. Senior vs. Staff Differentiators

| Topic | Senior answer | Staff answer |
|---|---|---|
| Keyword search | "Use Elasticsearch." | "ES fits most companies; here's why like-driven reindexing and trillion-doc cost push us to a custom index — and what we give up." |
| Likes | Update the index on every like. | Quantifies 14M ops/s; introduces windowed aggregation + milestone updates + re-rank. |
| Sharding | "Shard the index." | "Sharding choice decides which query fans out. Term-partitioning makes single-term reads O(1 shard) and moves cost to async writes." |
| Storage | "Add more nodes." | Shows ~19 PB RAM math → hot/cold tiering, caps, compression. |
| Multi-keyword | "Intersect lists." | Smallest-first intersection, merge-join with early stop, query-log-driven bigrams, and the phrase-vs-AND semantic difference. |
| Correctness | — | Deletion visibility as the CP exception; tokenizer parity; double-precision score trap; idempotent writes for at-least-once. |
| Operations | — | Consumer lag as freshness SLI; rebuild-from-source; blue/green index versions. |
| Evolution | — | Flags that adding privacy filtering breaks CDN/result caching. |

---

## 14. 45-Minute Interview Time Allocation

| Minutes | Phase |
|---|---|
| 0–4 | Functional reqs + explicit out-of-scope |
| 4–7 | NFRs, anchor constraint (likes mutate ordering) |
| 7–11 | Capacity math → "index doesn't fit in RAM", "likes = 14M ops/s" |
| 11–14 | Entities + API (idempotent PUT like, cursor pagination) |
| 14–25 | HLD: create, like, search, sort (naive → inverted index → two lists) |
| 25–40 | Deep dives: sharding/tiering, multi-keyword, hot queries, ingestion |
| 40–45 | Failure modes, CAP per component, evolution (privacy, ranking) |

---

## 15. Quick-Reference Cheatsheet

- **Numbers:** 35k posts/s, 350k likes/s, 35k searches/s peak; 3.65T posts; ~146T postings; naive RAM index ~19 PB → hot tier ~150 TB.
- **Index:** term → two lists: `recency` (score=createdAt) and `likes` (score=milestone count).
- **Likes:** PK (postId,userId) for idempotency → Redis INCR → Kafka → Flink window → milestone (2^k) index update → re-rank at query time with 4× over-fetch.
- **Shard:** term-partitioned + consistent hashing + hot-term sub-shards + time tiering.
- **Multi-keyword:** query-log bigrams → else smallest-first intersection / merge-join with early stop.
- **Fast hot queries:** Redis + CDN cache, single-flight, cursor pagination, never materialize all results.
- **Storage:** tiering, caps, delta+varint/Roaring compression, stopwords, ids-only postings.
- **CAP:** AP/EL everywhere; deletion visibility is the CP exception (hydration-time filter).
- **Traps:** dual-write (use outbox), tokenizer parity, Snowflake-as-double-score, cache stampede.

---

## 16. Related Guides

- **Twitter / Fan-out Architecture** — hot-key/celebrity patterns analogous to hot terms.
- **News Feed System** — Snowflake IDs, cursor pagination, feed caching.
- **Yelp (local business search & reviews)** — search index design with Elasticsearch trade-offs.
- **Apache Kafka** — partitioning, consumer lag, `acks` + `min.insync.replicas` durability contract.
- **Sharding Strategies** — term vs. document partitioning, consistent hashing.
- **CAP / PACELC Theorem** — per-component positioning.
- **Distributed Unique ID Generation** — Snowflake IDs and time ordering.
- **Leaderboard System** — Redis sorted sets, top-K maintenance.
- **Instagram** — like counting and hot-post handling.
- **Circuit Breaker Pattern** — protecting Search Service from degraded index shards.
