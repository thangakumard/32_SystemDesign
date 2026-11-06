# Yelp (Local Business Search & Reviews) — System Design Interview Guide
### Senior & Staff Engineer Level

---

## Table of Contents

1. [Problem Statement & Scope](#1-problem-statement--scope)
2. [Functional Requirements](#2-functional-requirements)
3. [Non-Functional Requirements](#3-non-functional-requirements)
4. [Capacity Estimation](#4-capacity-estimation)
5. [Core Entities](#5-core-entities)
6. [API / System Interface](#6-api--system-interface)
7. [Data Flow](#7-data-flow)
8. [High-Level Design](#8-high-level-design)
   - 8.1 [Overall Architecture](#81-overall-architecture)
   - 8.2 [FR1 — Search Businesses](#82-fr1--search-businesses-geo--text--filters)
   - 8.3 [FR2 — View Business Page](#83-fr2--view-business-page)
   - 8.4 [FR3 — Write / Edit / Delete Review](#84-fr3--write--edit--delete-review)
   - 8.5 [FR4 — Upload Photos](#85-fr4--upload-photos)
   - 8.6 [FR5 — Owner Claims & Edits Listing](#86-fr5--owner-claims--edits-listing)
   - 8.7 [FR6 — Autocomplete](#87-fr6--autocomplete)
   - 8.8 [FR7 — Map View](#88-fr7--map-view-bounding-box-search)
   - 8.9 [FR8 — User Profile Reviews](#89-fr8--user-profile-reviews)
   - 8.10 [Functional Requirements Coverage Matrix](#810-functional-requirements-coverage-matrix)
9. [Deep Dives](#9-deep-dives)
   - DD1 [Geospatial Index — All Options](#dd1-geospatial-index--all-options-compared)
   - DD2 [Search Index Sharding & Fan-out](#dd2-search-index-sharding--fan-out)
   - DD3 [Ranking Pipeline](#dd3-ranking-pipeline)
   - DD4 [Rating Aggregation Correctness](#dd4-rating-aggregation-correctness)
   - DD5 ["Open Now" — Timezones, DST, Overnight Hours](#dd5-open-now--timezones-dst-overnight-hours)
   - DD6 [Keeping Search in Sync (CDC)](#dd6-keeping-search-in-sync-with-the-source-of-truth)
   - DD7 [Review Trust & Safety](#dd7-review-trust--safety)
   - DD8 [Caching & Hot Businesses](#dd8-caching--hot-businesses)
   - DD9 [Latency Budget (p99 < 200 ms)](#dd9-meeting-the-latency-budget-p99--200-ms)
   - DD10 [Availability & Multi-Region](#dd10-availability--multi-region)
   - DD11 [Domain Correctness Traps](#dd11-domain-correctness-traps-pattern-class)
   - DD12 [Efficient, Up-to-the-Minute Average Ratings](#dd12-efficiently-calculating--updating-average-ratings-for-search)
   - DD13 [Concurrent Stats Updates Without Lost Updates](#dd13-making-the-synchronous-stats-update-safe-under-concurrent-writes)
   - DD14 [One Review Per User Per Business](#dd14-guaranteeing-one-review-per-user-per-business)
   - DD15 [Atomic Review Insert + Stats Update](#dd15-keeping-the-review-insert-and-the-stats-update-atomic)
   - DD16 [Making Complex Queries More Efficient](#dd16-making-complex-queries-more-efficient)
   - DD17 [Searching by Predefined Location Names](#dd17-searching-by-predefined-location-names-cities-neighborhoods)
10. [CAP / PACELC Positioning](#10-cap--pacelc-positioning)
11. [Failure Modes & Mitigations](#11-failure-modes--mitigations)
12. [Senior vs Staff Answer Differentiators](#12-senior-vs-staff-answer-differentiators)
13. [Interview Time Allocation](#13-interview-time-allocation)
14. [Quick-Reference Cheatsheet](#14-quick-reference-cheatsheet)
15. [Related Guides](#15-related-guides)

---

## 1. Problem Statement & Scope

Yelp lets users **discover local businesses** ("ramen near me, open now, $$, 4+ stars"), view business pages, and **read and write reviews with star ratings and photos**. Business owners claim and maintain their listings.

### How this differs from the Proximity Service guide

The Proximity Service guide answers *"which points lie within radius R of me?"* That is roughly **20% of Yelp**. The rest — where Staff candidates separate themselves — is:

- Combining **geo + free text + structured filters + time ("open now") + ranking** in one low-latency query
- Keeping **ratings exact** under edits, deletions, and fraud removal
- **Trust & safety** as a first-class pipeline
- **Index freshness** — how quickly a new review or an owner's edit shows up in search

### What the interviewer is really testing

- Can you design a **read-heavy, latency-sensitive, multi-dimensional search** system?
- Do you see that **the shard key decides which query becomes the expensive fan-out**?
- Can you keep **derived data (ratings, search index) consistent** with its source of truth?
- Do you treat "open now", review bombing, and geo edge cases as **correctness problems**?

---

## 2. Functional Requirements

| # | Requirement | Notes |
|---|---|---|
| **FR1** | Search businesses by location + keyword + filters | Filters: category, price, min rating, open now, distance, attributes |
| **FR2** | View business page | Details, hours, photos, rating + histogram, paginated sorted reviews |
| **FR3** | Write / edit / delete a review with 1–5 stars | **One review per user per business** |
| **FR4** | Upload photos | For a business, optionally attached to a review |
| **FR5** | Owner claims and edits listing; responds to reviews | Hours, special hours, photos, attributes |
| **FR6** | Autocomplete in the search box | Business names + categories, location-aware |
| **FR7** | Map view | Search inside a bounding box as user pans/zooms |
| **FR8** | View a user's profile with their reviews | Secondary access pattern |

**Out of scope (say it explicitly):** reservations, ordering, messaging, ads auction internals (ads are a ranking input only), recommendations feed.

---

## 3. Non-Functional Requirements

| Requirement | Target | Drives |
|---|---|---|
| Search latency | **p99 < 200 ms** server-side | DD2, DD3, DD9 |
| Business page latency | p99 < 100 ms | DD8 |
| Availability (reads) | 99.99% | DD10 |
| Scale | 200M businesses, 100M DAU | §4 |
| Read:write ratio | Several hundred : 1 | Read-optimized projections |
| Search freshness | New review → rating in search within seconds–minutes; owner edits within ~1 min | DD6 |
| Read-your-writes | Author sees own review immediately | §8.4 |
| Rating correctness | Displayed aggregate converges **exactly** to underlying recommended reviews | DD4 |
| Abuse resistance | Fake reviews, review bombing, scraping | DD7 |
| Privacy | Strip photo EXIF GPS; no exposure of reviewer location | §8.5 |

---

## 4. Capacity Estimation

> Numbers are **illustrative** — chosen to drive decisions, not claimed as Yelp's real figures. Decimal (SI) units throughout.

```
Businesses: 200M      DAU: 100M

SEARCH (client-facing)
  5 searches/DAU/day          → 500M/day
  500,000,000 / 86,400        ≈ 5,800 QPS avg
  Peak (3×, dinner hours)     ≈ 17,400 QPS

BUSINESS PAGE VIEWS
  10/DAU/day → 1B/day         ≈ 11,600 QPS avg  → ~35,000 QPS peak

AUTOCOMPLETE
  ~4 keystroke requests/search → ~70,000 QPS peak (mostly cache hits)

WRITES
  Reviews: 2M/day             ≈ 23/s avg, ~70/s peak
  Photos:  2M/day             ≈ 23/s avg
  Owner edits: 200K/day       ≈ 2/s
  Read:write                  ≈ 1.5B / 4.2M ≈ 360 : 1

STORAGE
  Business record   200M × 2 KB      = 400 GB
  Reviews existing  1B × 1 KB        = 1 TB;  growth 2 GB/day ≈ 0.73 TB/yr
  Photos            2M × 3 MB/day    = 6 TB/day ≈ 2.2 PB/yr (original + variants)

GEO DATA ALONE
  200M × (8B id + 16B lat/lng + 8B cell) ≈ 6.4 GB  → fits in RAM on one node

SEARCH INDEX
  200M docs × ~5 KB = 1 TB primary → ~25 shards of ~40 GB; 2 replicas → ~3 TB

INTERNAL FAN-OUT (sizes the search cluster)
  Doc-id sharded:  25 shards × 17,400  ≈ 435,000 shard-queries/s peak
  Geo-partitioned: ~1.2 shards/query   ≈  21,000 shard-queries/s peak  (~20× less)
```

**Takeaways:**
1. **Geo data is tiny.** Location indexing is not the scaling problem; the *combined* query is.
2. **Photos dominate storage, search dominates compute** — separate systems.
3. **Writes are ~70/s.** We can afford **strong consistency** on the review path.
4. **Internal fan-out, not client QPS, sizes the search tier.**

---

## 5. Core Entities

| Entity | Key fields | Source of truth | Partition key | Notes |
|---|---|---|---|---|
| **Business** | business_id, name, lat, lng, s2_cell, address, timezone (IANA), categories[], price_tier, attributes, status, owner_user_id, version | PostgreSQL | business_id | `status`: active / temporarily_closed / permanently_closed |
| **BusinessHours** | business_id, day_of_week, open_local, close_local | PostgreSQL | business_id | close < open ⇒ overnight |
| **SpecialHours** | business_id, local_date, open_local, close_local (NULL = closed) | PostgreSQL | business_id | Holidays, temporary closures |
| **Review** | review_id, business_id, user_id, rating, body, status, version, created_at | PostgreSQL | **business_id** | UNIQUE(business_id, user_id); status: recommended / not_recommended / removed |
| **BusinessStats** | business_id, review_count, rating_sum, histogram[5] | PostgreSQL | business_id | Co-located with reviews; counts recommended only |
| **OwnerResponse** | review_id, business_id, body, created_at | PostgreSQL | business_id | One per review |
| **Photo** | photo_id, business_id, review_id?, uploader_id, s3_key, variants, status, quality_score | PostgreSQL (metadata) + S3 (bytes) | business_id | status: uploading / processing / ready / rejected |
| **User** | user_id, name, home_city, trust_score, created_at | PostgreSQL | user_id | |
| **UserReviewIndex** (projection) | user_id, review_id, business_id, created_at | PostgreSQL / Cassandra | user_id | Derived via CDC for FR8 |
| **SearchDocument** (projection) | business_id, text fields, geo_point, filters, open_intervals[], rating, bayesian_rating, review_count, features | Elasticsearch | region → shard | Derived via CDC |
| **OwnerClaim** | claim_id, business_id, user_id, verification_method, state | PostgreSQL | business_id | Verification state machine |

```
User ─1───*─ Review ─*───1─ Business ─1───1─ BusinessStats
  │              │               │
  │              └─0..1─ OwnerResponse
  └─1───*─ Photo ─*───────1─────┘
                  Business ─1───*─ BusinessHours / SpecialHours
```

**Design principle:** PostgreSQL holds every source-of-truth entity. `SearchDocument`, `UserReviewIndex`, caches, and the feature store are **projections**, rebuildable from the source.

---

## 6. API / System Interface

REST over HTTPS, JSON, OAuth2 bearer tokens. All list endpoints use **opaque cursors**, never offsets.

### 6.1 Search (FR1, FR7)

```http
GET /v1/search
    ?q=ramen
    &lat=47.61&lng=-122.20&radius_m=5000          # circle search
    # OR &bbox=47.55,-122.35,47.70,-122.10        # map view (FR7)
    &categories=japanese,noodles
    &price=2,3
    &min_rating=4
    &open_now=true                                # or &open_at=2026-10-31T19:30:00Z
    &sort=best_match|distance|rating|review_count
    &limit=20
    &cursor=eyJzIjo0LjMyLCJpZCI6MTIzfQ

200 OK
{
  "results": [
    {
      "business_id": "b_123",
      "name": "Menya Ramen",
      "rating": 4.6, "review_count": 1832,
      "price_tier": 2,
      "distance_m": 840,
      "is_open_now": true, "closes_at_local": "22:00",
      "categories": ["japanese", "ramen"],
      "cover_photo_url": "https://cdn.../p_88/600.webp",
      "snippet": "...the tonkotsu broth is..."
    }
  ],
  "next_cursor": "...",
  "partial": false            # true if a shard timed out (graceful degradation)
}
```

### 6.2 Business (FR2, FR5)

```http
GET    /v1/businesses/{business_id}
GET    /v1/businesses/{business_id}/reviews?sort=newest|highest|lowest|relevance&cursor=...
GET    /v1/businesses/{business_id}/photos?cursor=...

PATCH  /v1/businesses/{business_id}                    # owner only
       If-Match: "v42"                                  # optimistic concurrency → 412 on conflict
       { "hours": [...], "special_hours": [...], "attributes": {...} }

POST   /v1/businesses/{business_id}/claims             # start ownership claim
POST   /v1/reviews/{review_id}/response                # owner reply
```

### 6.3 Reviews (FR3, FR8)

```http
POST   /v1/businesses/{business_id}/reviews
       Idempotency-Key: 7f3c...                        # safe client retries
       { "rating": 4, "body": "...", "photo_ids": ["p_1"] }
       → 201 Created  { "review_id": "r_9", "status": "published" }
       → 409 Conflict if user already reviewed this business

PUT    /v1/reviews/{review_id}      If-Match: "v3"    { "rating": 5, "body": "..." }
DELETE /v1/reviews/{review_id}
GET    /v1/users/{user_id}/reviews?cursor=...          # FR8
```

### 6.4 Photos (FR4)

```http
POST /v1/photos/upload-intent
     { "business_id": "b_123", "content_type": "image/jpeg", "size_bytes": 3100000 }
     → { "photo_id": "p_88", "upload_url": "https://s3...presigned", "expires_in": 600 }

# client PUTs bytes directly to upload_url, then:
POST /v1/photos/{photo_id}/complete        → 202 Accepted (processing)
```

### 6.5 Autocomplete (FR6)

```http
GET /v1/autocomplete?prefix=ram&lat=47.61&lng=-122.20&limit=8
→ { "suggestions": [ {"type":"category","text":"Ramen"},
                     {"type":"business","text":"Ramen Danbo","business_id":"b_77"} ] }
```

### 6.6 Internal interfaces

```
Search Svc → Ranking Svc (gRPC)
  Rank(query_ctx, user_ctx, candidates[≤500 {business_id, stage1_score, distance}])
    → ranked[20]          deadline: 50 ms

CDC events (Kafka, keyed by business_id)
  business.changed  { business_id, table, op, lsn, version }
  review.changed    { business_id, review_id, op, old_status, new_status, lsn }
  stats.changed     { business_id, lsn }
  photo.ready       { business_id, photo_id }
```

**API design choices worth calling out:**

| Choice | Why |
|---|---|
| `Idempotency-Key` on review POST | Mobile retries must not create duplicate reviews or double-count stats |
| `If-Match` / version on PUT/PATCH | Prevents lost updates between owner's web and mobile sessions |
| Opaque cursors | Lets us change pagination strategy (keyset, `search_after`, PIT) without breaking clients |
| `partial: true` flag | Search returns degraded results instead of failing when a shard is slow |
| `open_at` parameter | Same mechanism as `open_now`; supports "open Friday at 8 pm" |

---

## 7. Data Flow

### 7.1 Read path — search (FR1)

```
1. Client → API Gateway        auth, rate limit, attach geo/IP context
2. Gateway → Search Svc
3. Search Svc: query understanding  (spell-correct, "cheap" → price=1, synonyms, category map)
4. Search Svc: check search cache   key = (S2 cell L13, normalized query, filters, time bucket)
       hit  → step 8
5. Search Svc: S2 cover of circle/bbox → which region indexes to query
6. ES (region shards): filter (geo, price, rating, open interval) + BM25 + distance decay
       → top ~500 candidates
7. Ranking Svc: fetch features (feature store) → model scores → top 20
8. Hydrate display fields (cache) · recompute per-user distance
9. Return results + cursor;  async: log impression for ranking training
```

### 7.2 Read path — business page (FR2)

```
1. Client → Gateway → Business Svc
2. Redis: biz:{id} (details+hours) and stats:{id}     (hit rate high: power-law traffic)
       miss → Postgres replica → populate cache (single-flight)
3. Review Svc: first page of reviews (keyset on business shard; cached for hot businesses)
4. Photos: metadata from cache; bytes served by CDN
```

### 7.3 Write path — review (FR3)

```
1. Client → Gateway → Review Svc  (Idempotency-Key checked)
2. Postgres business shard, ONE transaction:
     INSERT review (UNIQUE business_id,user_id)
     UPDATE business_stats (+1 count, +rating sum, histogram)
3. Commit → 201 to client; session flag "read-from-primary" for ~30 s (read-your-writes)
4. WAL → Debezium → Kafka review.changed / stats.changed
5. Consumers (independent, at-least-once, idempotent):
     a. Cache invalidator  → delete biz/stats/review-page keys
     b. Search indexer     → re-read business state → upsert ES doc (external version)
     c. UserReviewIndex    → upsert (user_id, review_id)
     d. Fraud scorer       → may flip status → step 2-style transaction applies stats delta
     e. Notifications      → notify owner of new review
```

### 7.4 Write path — photo (FR4)

```
upload-intent → presigned S3 PUT → S3 event → image workers
  (validate, strip EXIF, resize, moderate, perceptual-hash dedup)
  → photo.status = ready → CDC → cache invalidation + ES (photo count, cover photo)
```

### 7.5 Write path — owner edit (FR5)

```
PATCH (If-Match) → Business Svc → Postgres (version check)
  → sensitive fields (name, address, location) → moderation queue, not applied directly
  → CDC → indexer regenerates open_intervals + doc → cache invalidation
```

---

## 8. High-Level Design

### 8.1 Overall Architecture

```
                         ┌──────────────────────────────┐
   Mobile / Web ────────▶│ CDN  (photos, static assets) │
        │                └──────────────────────────────┘
        ▼
 ┌──────────────────────┐
 │ API Gateway / LB     │  authN · rate limiting · bot detection · routing
 └──────────┬───────────┘
            │
 ┌──────────┼──────────────┬───────────────┬───────────────┬──────────────┐
 ▼          ▼              ▼               ▼               ▼              ▼
┌─────────┐┌────────────┐┌─────────────┐┌─────────────┐┌────────────┐┌───────────┐
│ Search  ││ Ranking    ││ Business    ││ Review      ││ Media      ││ Autocomp. │
│ Svc     ││ Svc        ││ Svc         ││ Svc         ││ Svc        ││ Svc       │
│ parse,  ││ ML rerank  ││ listings,   ││ write/edit, ││ presigned  ││ prefix    │
│ route,  ││ (gRPC)     ││ hours,      ││ pagination, ││ upload     ││ top-K     │
│ retrieve│└─────┬──────┘│ claims      ││ stats delta ││            ││           │
└───┬─────┘      │       └─────┬───────┘└──────┬──────┘└─────┬──────┘└─────┬─────┘
    │            ▼             │               │             │             │
    │     ┌─────────────┐      │               │             │             │
    │     │Feature Store│      │               │             │             │
    │     │ (Redis)     │      │               │             │             │
    │     └─────────────┘      ▼               ▼             ▼             ▼
    │              ┌──────────────────────────────────┐  ┌─────────┐ ┌──────────┐
    │              │ PostgreSQL — sharded by          │  │ S3/GCS  │ │ Redis    │
    │              │ business_id (source of truth)    │  │ photos  │ │ prefix   │
    │              │ businesses · hours · reviews ·   │  └────┬────┘ │ top-K    │
    │              │ business_stats · photos meta     │       │      └──────────┘
    │              └───────────────┬──────────────────┘       │
    ▼                              │ WAL / CDC (Debezium)     │ S3 events
┌───────────────────┐              ▼                          ▼
│ Elasticsearch     │      ┌───────────────────┐     ┌──────────────────┐
│ geo-partitioned   │◀─────│ Kafka             │◀────│ Image workers    │
│ region indexes    │      │ business.changed  │     │ resize · moderate│
└───────────────────┘      │ review.changed    │     │ EXIF strip       │
        ▲                  │ stats.changed     │     └──────────────────┘
        │                  └──┬──────┬──────┬──┘
   Indexer consumers ─────────┘      │      │
                       ┌─────────────┘      └───────────────┐
                       ▼                                    ▼
             ┌───────────────────┐  ┌──────────────┐  ┌────────────────────┐
             │ Cache invalidator │  │ Fraud scorer │  │ UserReviewIndex     │
             │ → Redis           │  │ + mod queue  │  │ projection (FR8)    │
             └───────────────────┘  └──────────────┘  └────────────────────┘

 Redis cluster: business page cache · stats cache · popular search cache · sessions
 etcd/ZooKeeper: versioned region map (S2 cells → search index)
 Elasticsearch query per region shard: BM25 text match ∩ geo_point filter ∩ price/rating/open-interval filters
```

**Shape of the system:** one source of truth (Postgres) and many **CDC-fed projections** (search index, profile index, caches, features). Every consistency question becomes "how far does projection X lag, and how do we detect and repair drift?"

---

### 8.2 FR1 — Search Businesses (geo + text + filters)

#### Talk Track (120–180 sec): "How will users be able to search for businesses?"

> Here's how a user search actually resolves end to end. A user types something like "ramen near me, open now, 4 stars and up." That request goes through the API gateway — which handles auth, rate limiting, and attaches geo context — and lands at the Search Service.
>
> The Search Service first does query understanding: spell-correction, mapping "cheap" to a price filter, synonym expansion, category mapping — "ramen" becomes `category=japanese,noodles` under the hood.
>
> Before touching anything expensive, it checks a Redis search cache. The key isn't the user's raw lat/lng — that would be unique per user and never hit — it's the user's location snapped to a coarse S2 cell, plus the normalized query, filters, and a time bucket for "open now." That snapping is S2's whole job here: give nearby users the same key so they share a cache entry. Most head queries in dense areas repeat within a short TTL, so this cache absorbs a lot of traffic.
>
> On a miss, we take the user's circle or bounding box and compute an S2 cover — the set of S2 cells intersecting that area. That cover tells us which region shards in Elasticsearch to query; we don't fan out to every shard, just the one or two regions the user is actually in. That's the single biggest lever on tail latency — fewer shards per query means p99 isn't dragged down by the slowest of many.
>
> Inside those shards, Elasticsearch applies hard filters — geo distance, price tier, rating, the precomputed "open now" interval, business status — intersected with a BM25 text match on name and category, combined with distance decay and log of review count. That retrieval stage returns roughly 500 candidates in under 50 ms.
>
> Those candidates go to a separate Ranking Service, which pulls features from a feature store — text relevance, distance, Bayesian-adjusted rating, personalization signals — and reranks down to the top 20 with a small model. If that reranker is slow or down, a circuit breaker falls back to the Stage-1 order, so a ranking-model outage never breaks search.
>
> Finally we hydrate display fields like photos and snippets from cache, recompute exact distance per user, and return the page with an opaque cursor for pagination — logging an impression asynchronously for ranker training.
>
> So the short version: cache, then geo-partitioned routing, then filtered BM25 retrieval, then an ML rerank — each stage with its own latency budget, adding up to under 200 ms at p99.

```
Client ──▶ Gateway ──▶ Search Svc ──┬──▶ Redis (search cache) ──hit──▶ return
                                    │
                                    ├──▶ Region map (S2 cell cover → which region indexes)
                                    │
                                    ├──▶ Elasticsearch region shard(s)
                                    │       BM25 text match (name, category)
                                    │       ∩ geo_point filter (distance)
                                    │       ∩ price / rating / open-interval filters
                                    │    → 500 candidates
                                    │
                                    └──▶ Ranking Svc ──▶ Feature Store  → top 20
```

**Options for evaluating the multi-dimensional query:**

| Option | Description | Pros | Cons |
|---|---|---|---|
| A. Separate geo service ∩ text service | Geo returns IDs in radius; text returns IDs matching "ramen"; intersect | Specialized services; geo reusable | "Restaurants in NYC" → 50K IDs each side; network intersection; filters need a third system |
| **B. Unified search engine (ES/OpenSearch)** | One query: `filter[geo_distance, price, rating, open_interval] + must[match]` | Single pass; filters are cached bitsets; distance sort built in | ES ops cost; complex ML must be a separate rerank stage |
| C. Precomputed top-N per (S2 cell, category) | Offline materialization | O(1) serving for browse tabs | No free text, no arbitrary filter combos; stale |
| D. Postgres + PostGIS + full-text (tsvector) | Everything in source DB | One system; transactional freshness | Poor relevance tuning; scaling reads on OLTP DB; no BKD-tree geo + BM25 combo at this QPS |

**Choose B, with C as a cache for top browse queries.** Details: DD1 (geo index), DD2 (sharding), DD3 (ranking).

**Sparse areas:** use a **density-aware default radius** precomputed per S2 cell; if results < threshold, expand once. Avoids iterative doubling round-trips.

**Pagination:** `search_after` on `(score, business_id)` encoded in cursor; cap ~50 pages (also limits scrapers). Optional point-in-time snapshot for stable ordering within a session.

---

### 8.3 FR2 — View Business Page

#### Talk Track (120–180 sec): "How will users be able to view a business's details and reviews?"

> When a user opens a business page, the client calls the Gateway, which routes to the Business Service. This is a read-heavy, latency-sensitive path — p99 target is under 100 ms — so almost nothing here touches Postgres directly on the happy path.
>
> First, the Business Service looks up two Redis keys: `biz:{id}:v{version}` for the listing details — name, address, hours, attributes, cover photo — and `stats:{id}` for the rating and review count. Because business traffic follows a power law, a small fraction of businesses get a large fraction of views, so this cache has a very high hit rate. On a miss, it falls through to a Postgres read replica, and uses single-flight so a cache-stampede on a newly-popular business doesn't send a hundred concurrent requests to the database for the same row — one request loads it, the rest wait on that one result.
>
> The rating itself is never computed live with an `AVG()` query. It's read directly off `business_stats`, a row that's kept exact via same-transaction deltas every time a review is created, edited, deleted, or flipped by fraud detection — so what the user sees is both fast and exactly correct, not an approximation.
>
> In parallel, the Review Service fetches the first page of reviews. For hot businesses that's also cached — `reviews:{id}:{sort}:p1` — and on a miss it reads from the business's Postgres shard using keyset pagination: `WHERE business_id = ? AND review_id < :cursor ORDER BY review_id DESC LIMIT 20`, never OFFSET, so paging stays O(1) regardless of how deep the user scrolls. If the user sorts by "relevance" instead of "newest," we're not scoring on read — each review already carries a precomputed quality score, based on length, helpful votes, and author trust, so relevance sort is just another indexed sort order.
>
> Photos are handled separately: metadata comes from cache, but the actual bytes are served by the CDN, so the app tier never proxies image traffic.
>
> One detail worth calling out: the "open now" badge on this page is computed on the fly from the business's hours and timezone at read time — that's fine because it's one business, not a search result list of hundreds, so the cost of a per-request timezone computation here is negligible. That's different from search, where "open now" has to be precomputed into UTC intervals ahead of time because it's evaluated across every candidate at once.
>
> And if the viewer just posted a review themselves, read-your-writes matters: their session is flagged to read from the primary for about 30 seconds, so they see their own review immediately instead of hitting a stale cached page.

```
Client ──▶ Gateway ──▶ Business Svc ──▶ Redis biz:{id}:v{ver}, stats:{id}
                           │                 │ miss (single-flight)
                           │                 ▼
                           │          Postgres read replica (business shard)
                           │
                           ├──▶ Review Svc ──▶ Redis reviews:{id}:newest:p1  (hot businesses)
                           │                     │ miss
                           │                     ▼
                           │              Postgres business shard (keyset pagination)
                           │
                           └──▶ photo metadata (cache) ; bytes via CDN
```

| Decision | Options | Choice & why |
|---|---|---|
| Rating shown | Live AVG() vs precomputed stats | Precomputed `business_stats` (exact, see DD4) |
| Review sort "relevance" | Compute on read vs precompute score | Precompute per-review quality score (length, votes, author trust) into an indexed column; sort via index |
| Pagination | OFFSET vs keyset | Keyset: `WHERE business_id=? AND review_id < :cursor ORDER BY review_id DESC LIMIT 20` |
| "Open now" badge | Store boolean vs compute | Compute from hours + timezone in Business Svc at read time (cheap for one business) |

---

### 8.4 FR3 — Write / Edit / Delete Review

#### Talk Track (120–180 sec): "How will users be able to write, edit, and delete a review?"

> A review write is low-volume — maybe 70 a second at peak — but it has strict correctness requirements: one review per user per business, and an aggregate rating that must stay exactly right. So unlike search, I'm willing to pay for strong consistency here.
>
> The client submits with an `Idempotency-Key` header. Mobile networks retry requests, and a retried review POST must not create a duplicate or double-count the stats, so the Review Service checks a Redis idempotency store — key to cached response, 24-hour TTL — before doing anything else. If this key's been seen, it just replays the original response.
>
> The actual write is a single Postgres transaction, on the shard owned by `business_id`: insert the review row, which has a `UNIQUE(business_id, user_id)` constraint so a duplicate review fails fast at the database level, and in the same transaction update `business_stats` — count +1, rating sum + r, histogram bucket +1. That's the key design decision: because reviews are sharded by `business_id`, not `user_id` or `review_id`, both the uniqueness check and the stats update are local to one shard. No distributed transaction, no two-phase commit — just one ACID transaction on one node. The tradeoff is that fetching one user's full review history now fans out across shards, but that's a secondary, lower-traffic query, so I built a projection for it instead of paying the fan-out cost on the hot path.
>
> Once that commits, we return 201 immediately, and flag the user's session to read from the primary for about 30 seconds — that's the read-your-writes guarantee, so the author sees their own review on the business page right away instead of hitting a stale cache or replica.
>
> From there it's all asynchronous. The transaction's WAL is picked up by Debezium and published to Kafka as `review.changed` and `stats.changed`, keyed by `business_id` so events for the same business stay ordered. Five independent consumers react: a cache invalidator clears the business/stats/review-page keys; the search indexer re-reads the business's full state and upserts the Elasticsearch document; a `UserReviewIndex` projection keyed by `user_id` gets the new `(review_id, business_id)` pair so profile lookups stay single-shard; a fraud scorer evaluates the review asynchronously and can flip its status; and a notification goes to the business owner.
>
> Edits and deletes go through the exact same transactional path — same idempotency check, same shard, same delta-to-stats pattern, just with a computed delta instead of a flat +1 (for example, an edit from 3 stars to 5 stars adds +2 to the sum and shifts the histogram). Nothing about the write path changes; only the arithmetic does.

```
Client ──▶ Gateway ──▶ Review Svc ──▶ Idempotency store (Redis, key → response, 24 h)
                           │
                           ▼
            ┌──────────────────────────────────────────┐
            │ Postgres shard for business_id           │
            │ BEGIN                                    │
            │   INSERT reviews  (UNIQUE biz,user)      │
            │   UPDATE business_stats (delta)          │
            │ COMMIT                                   │
            └──────────────────┬───────────────────────┘
                               │ WAL
                               ▼
                       Debezium → Kafka review.changed
                               │
         ┌──────────┬──────────┼──────────────┬───────────────┐
         ▼          ▼          ▼              ▼               ▼
     Cache inv.  Indexer   UserReviewIdx  Fraud scorer   Notifications
```

**Review sharding options:**

| Shard key | Pros | Cons |
|---|---|---|
| **business_id** | Business page = 1 shard; **UNIQUE(business_id, user_id) is a local constraint**; stats update is a **local transaction** | User profile (FR8) fans out → solved with projection |
| user_id | Profile = 1 shard | Hot path (business page) fans out; uniqueness and stats become distributed |
| review_id | Even spread | Both main queries fan out |

**Choose business_id.** The dominant query wins the shard key; the secondary query gets a projection.

**Read-your-writes options:**

| Option | Pros | Cons |
|---|---|---|
| **Session flag → read from primary for ~30 s** | Simple, exact | Small extra primary load (writes are rare) |
| Client merges its own review into the page | Zero server cost | Breaks across devices |
| Wait until replica LSN ≥ write LSN | Exact, no primary load | More plumbing |

**Choose the session flag.**

**Edits and deletes** go through the same transaction with a computed delta (DD4). `Idempotency-Key` makes client retries safe.

---

### 8.5 FR4 — Upload Photos

```
Client ──(1) upload-intent──▶ Media Svc ──▶ Postgres photo(status=uploading)
   │                              │
   │◀──── presigned PUT URL ──────┘
   │
   └──(2) PUT bytes ─────────────▶ S3 (originals bucket)
                                      │ (3) S3 event
                                      ▼
                                 Kafka photo.uploaded
                                      │
                                      ▼
                              Image worker pool
                     validate · EXIF/GPS strip · malware sniff
                     resize 150/600/1200 (WebP/AVIF) · moderation model
                     perceptual hash (dedup) · quality score
                                      │
                      ┌───────────────┴───────────────┐
                      ▼                               ▼
              S3 (variants bucket)        Postgres photo(status=ready)
                      │                               │ CDC
                      ▼                               ▼
                     CDN                   cache invalidation · ES cover photo
```

| Decision | Options | Choice & why |
|---|---|---|
| Upload path | Through app servers vs presigned direct | **Presigned** — app tier doesn't proxy 6 TB/day |
| Variants | On-the-fly resize vs pre-generate | **Pre-generate** — extreme read:write, lower read latency |
| Moderation | Sync vs async | **Async**; photo hidden until ready |
| Storage tiering | All Standard vs lifecycle | Non-cover photos → Infrequent Access after 90 days (2.2 PB/yr) |

---

### 8.6 FR5 — Owner Claims & Edits Listing

```
Owner ──▶ Gateway ──▶ Business Svc
                         │
              ┌──────────┴───────────────┐
              ▼                          ▼
   Claim workflow                 PATCH listing (If-Match version)
   (phone/email/doc verify)              │
   state machine in Postgres     ┌───────┴────────────────────┐
                                 ▼                            ▼
                      non-sensitive fields          sensitive fields
                      (hours, attributes)           (name, address, lat/lng)
                      → apply directly              → moderation queue
                                 │                            │
                                 └──────────┬─────────────────┘
                                            ▼
                                  Postgres → CDC → indexer
                                  (regenerate open_intervals) → cache invalidation
```

- **Optimistic concurrency** via `version` → `412 Precondition Failed` on conflict.
- **Moderation for sensitive fields** prevents hijacking a competitor's listing or moving it.
- **Owner responses** stored as `OwnerResponse` on the business shard; displayed under the review.

---

### 8.7 FR6 — Autocomplete

```
Client (debounce ~100 ms, cancel in-flight)
   ──▶ Gateway ──▶ Autocomplete Svc
                      ├── prefix ≤ 3 chars → Redis  ac:{region}:{prefix}  (top-K precomputed)
                      └── prefix > 3 chars → ES completion suggester with geo context
```

| Option | Pros | Cons |
|---|---|---|
| ES completion suggester / edge n-grams | Reuses search infra; geo context | Heavier per request |
| In-memory trie per region | Fastest | Build/deploy pipeline; memory per region |
| Precomputed top-K per prefix in Redis | O(1) | Explodes for long prefixes |

**Choose hybrid**: Redis for short prefixes (where most traffic and cache hits are), ES for longer ones. Top-K rebuilt nightly from query logs + business popularity per region.

---

### 8.8 FR7 — Map View (bounding-box search)

```
Client map pan/zoom (debounce ~300 ms)
   ──▶ Search Svc  (same pipeline as FR1, geo_bounding_box filter)
         · snap viewport to S2 cells → cache reuse across overlapping viewports
         · zoomed out (> N results) → return clustered pins (pre-aggregated per S2/H3 cell)
```

| Concern | Choice |
|---|---|
| Too many pins at low zoom | Server-side **clustering** by cell with counts (H3 is good here) |
| Request storms while panning | Client debounce + cell-snapped cache keys |
| Ranking within viewport | Same ranker; distance feature replaced by "distance to viewport center" |

---

### 8.9 FR8 — User Profile Reviews

```
Client ──▶ Gateway ──▶ Review Svc ──▶ UserReviewIndex (sharded by user_id)
                                         → review_ids + business_ids
                                      ──▶ batch-get review bodies from business shards
                                          (parallel, bounded: 20 per page)
```

| Option | Pros | Cons |
|---|---|---|
| Scatter-gather all review shards by user_id | No extra storage | Fan-out to every shard on each profile view |
| **CDC projection keyed by user_id** | Single-shard lookup | Seconds of lag; extra storage (~40 B/review → 40 GB) |
| Duplicate full review into user-sharded table | Single read | Doubles review storage; two copies to keep in sync |

**Choose the projection** (IDs only) + bounded batch-get.

---

### 8.10 Functional Requirements Coverage Matrix

| FR | Primary components | Store(s) | Section |
|---|---|---|---|
| FR1 Search | Search Svc, Ranking Svc, Feature Store | Elasticsearch, Redis | 8.2, DD1–DD3, DD5 |
| FR2 Business page | Business Svc, Review Svc | Postgres, Redis, CDN | 8.3, DD8 |
| FR3 Reviews | Review Svc, Fraud scorer | Postgres (txn), Kafka | 8.4, DD4, DD7 |
| FR4 Photos | Media Svc, Image workers | S3, CDN, Postgres | 8.5 |
| FR5 Owner edits | Business Svc, Moderation | Postgres, Kafka → ES | 8.6, DD5, DD6 |
| FR6 Autocomplete | Autocomplete Svc | Redis, ES | 8.7 |
| FR7 Map view | Search Svc | Elasticsearch, Redis | 8.8 |
| FR8 Profile | Review Svc | UserReviewIndex projection | 8.9 |

---

## 9. Deep Dives

### DD1. Geospatial Index — All Options Compared

*(Internals in the Proximity Service guide; here, the decision.)*

| Option | How it works | Pros | Cons | Verdict |
|---|---|---|---|---|
| Lat/lng range in SQL (2 B-trees) | `lat BETWEEN … AND lng BETWEEN …` | Trivial | Two 1-D indexes intersect poorly; box ≠ circle | Prototype only |
| **Geohash** | Interleaved bits → base-32 prefix | Any KV/B-tree; easy prefix sharding; cacheable keys | Fixed grid ⇒ uneven density; 9-cell edge queries; pole distortion | Good cache/routing key |
| **Quadtree (in-memory)** | Split cell until ≤ K points | Adapts to density; great k-NN | Per-server rebuild; update handling; startup time | Pure proximity services |
| **Google S2** | Sphere → cube → Hilbert curve → 64-bit cell IDs | No pole/antimeridian issues; covers any region with few cells; sortable ints | Library complexity | **Partitioning & cache key** |
| **Uber H3** | Hierarchical hexagons | Uniform neighbor distance; ideal for aggregation | Imperfect parent/child tiling | Map clustering, analytics |
| R-tree / PostGIS GiST | Bounding rectangles | Exact; polygon queries | Single-node; weak text combination | Admin/polygon queries |
| **ES `geo_point` (BKD tree)** | Block KD-tree per segment | **Composes with BM25 + filters + scoring in one pass** | Cluster ops; reindex discipline | **Query evaluation** |

**Decision:** ES `geo_point` evaluates the predicate; **S2 cells** decide partitioning and cache keys; H3 for map clustering.

> *"Geo data is 6 GB — any index handles location alone. I choose by composability with text and filters, and I separate query evaluation (BKD) from partitioning (S2)."*

---

### DD2. Search Index Sharding & Fan-out

The **shard key determines which query becomes the expensive fan-out.**

| Strategy | Pros | Cons |
|---|---|---|
| **Doc-id hash** `hash(business_id) % N` | Even storage and indexing load | Every query hits all N shards → **435K shard-queries/s** at peak; tail latency = slowest of N |
| **Pure geo partition** (S2 region per shard) | Query touches 1–2 shards | Hot metros (NYC, Tokyo) skew; growth needs rebalancing |
| **Hybrid (choose)** | Regions built from S2 cells, balanced by doc count *and* query load; within region, doc-hash over a few shards; hot regions get more shards/replicas | Routing layer and versioned region map |

```
Region map (versioned in etcd; cached in Search Svc)
  r_seattle  = {S2 cells…} → index biz_r_seattle   2 shards × 3 replicas
  r_nyc      = {S2 cells…} → index biz_r_nyc       8 shards × 6 replicas
  r_mt_west  = {S2 cells…} → index biz_r_mt_west   1 shard  × 2 replicas

Routing:  circle/bbox → S2 RegionCoverer → regions touched → query those → merge top-K
```

**Why it matters for tail latency:** with fan-out N, p99 of the query ≈ the p99.x of the slowest shard. Fewer shards per query ⇒ better p99 at the same per-shard performance.

**Region rebalancing:** create new region index, dual-index affected cells, swap region-map version atomically, drain old. Map version is included in cursors so pagination stays consistent across a swap.

---

### DD3. Ranking Pipeline

```
Stage 1 — Retrieval in ES (≤ 50 ms)          ~500 candidates
   BM25 × distance decay × log(review_count) × bayesian_rating prior
   + hard filters (geo, price, open interval, status = active)

Stage 2 — Re-rank (≤ 50 ms)                   top 20
   GBDT / small NN over features from feature store:
     text relevance · distance · bayesian rating · review count & recency
     CTR / conversion · photo quality · listing completeness
     personalization (user's categories, price tier) · owner responsiveness
   Ads blended by separate auction, labelled

Fallback — circuit breaker → Stage-1 order if reranker is slow/down
```

| Where ranking lives | Pros | Cons |
|---|---|---|
| All in ES (`function_score`, LTR plugin) | One hop | Model complexity limited; plugin deploys |
| **ES retrieval + separate reranker** | Any model; independent deploys & A/B tests | Extra hop; feature store needed |
| Static precomputed score | Cheapest | No relevance or personalization |

**Bayesian average** (rank on it, display raw mean):

```
bayesian = (C·m + Σr) / (C + n)        m = category mean (≈3.8), C = 20

1 review of 5★        → (76 + 5)    / 21    ≈ 3.86
2,000 reviews @ 4.7★  → (76 + 9400) / 2020  ≈ 4.69
```

**Feedback loop risk:** ranking on CTR favors already-top results (popularity bias). Mitigate with exploration slots and position-bias correction in training.

---

### DD4. Rating Aggregation Correctness

| Option | Correctness | Latency | Notes |
|---|---|---|---|
| Compute on read (`AVG`) | Exact | O(n) per view at 35K QPS | No |
| **Same-transaction counter update** (co-sharded) | **Exact, strongly consistent** | Instant | Possible because reviews & stats share business_id shard |
| Async Kafka aggregation | Eventual; drift if not idempotent | Seconds | Needed only if stats live elsewhere |
| Periodic batch recompute | Exact at batch time | Hours stale | **Use as reconciliation** |
| Redis INCR | Lossy on failover; no link to review txn | Instant | Permanent divergence risk |

**Choose same-transaction deltas + nightly reconciliation.**

| Event | Delta (applied in the review's transaction) |
|---|---|
| Create (recommended) | count +1, sum + r, hist[r] +1 |
| Edit r_old → r_new | sum + (r_new − r_old), hist shift |
| Delete | count −1, sum − r, hist[r] −1 |
| Fraud: recommended → not_recommended | count −1, sum − r |
| Fraud: not_recommended → recommended | count +1, sum + r |
| Business merge | Recompute from scratch (not delta) |

```sql
BEGIN;
  SELECT rating, status FROM reviews
   WHERE business_id = :b AND review_id = :r FOR UPDATE;   -- lock old state
  UPDATE reviews SET rating = :new, version = version + 1 WHERE ...;
  UPDATE business_stats
     SET rating_sum = rating_sum + (:new - :old), ...
   WHERE business_id = :b;
COMMIT;
```

- **Hot-row contention is a non-issue:** even a viral business gets ~0.3 reviews/s. The real risk is *review bombing* (DD7).
- **Reconciliation** recomputes SUM/COUNT per business nightly and alerts on any mismatch — expected drift is zero; the job proves it.
- **Search gets ratings via CDC** (seconds stale); **business page reads stats directly** (exact).

---

### DD5. "Open Now" — Timezones, DST, Overnight Hours

Complications: local-time hours vs UTC query; overnight spans (Fri 18:00–Sat 02:00); DST gaps/overlaps; holiday/special hours; searcher in a different timezone than the business.

| Approach | Pros | Cons |
|---|---|---|
| Script filter computing local time per candidate | Always correct | Per-doc script → slow at 17K QPS |
| `open_now` boolean refreshed by cron | Fast filter | 200M doc updates/min; index churn |
| **Precomputed UTC open intervals (next 7 days) as `date_range` field** | Fast `contains now` filter; overnight/DST/special hours resolved at generation | Daily regen job; must regen on hours edit, special-hours edit, and IANA tzdata release |

**Choose precomputed intervals**, generated with a real tz library (never fixed offsets). Also supports `open_at` queries for any time in the next 7 days.

```
Menya Ramen, America/Los_Angeles, Fri 11:00–23:00, Sat 11:00–01:00
→ open_intervals:
   [2026-10-30T18:00Z, 2026-10-31T06:00Z]
   [2026-10-31T18:00Z, 2026-11-01T08:00Z]   ← overnight; spans DST fall-back (Nov 1)
```

---

### DD6. Keeping Search in Sync with the Source of Truth

| Option | Pros | Cons |
|---|---|---|
| Dual write (DB then ES in app) | Simple | Partial failure ⇒ permanent divergence; races between concurrent writers |
| Transactional outbox | Atomic with DB write | Outbox table + relay to operate |
| **CDC from WAL (Debezium → Kafka)** | No app changes; captures admin SQL & backfills; per-key ordering | Row events need enrichment |
| Periodic full rebuild | Self-healing | Hours stale |

**Choose CDC + periodic full rebuild** (rebuild doubles as DR and schema-migration path).

```
Indexer (consumer group, topics keyed by business_id):
  1. Receive any change for business B
  2. Re-read FULL current state of B (business, hours, stats, cover photo)
  3. Build doc (incl. open_intervals, bayesian_rating)
  4. Index with version_type=external, version = source LSN/version
       → older events rejected → duplicates/out-of-order are harmless
  5. Commit offset after ES ack (at-least-once, idempotent)
```

**Coalescing:** a burst of 50 events for one business within 1 s → index once (per-key debounce in consumer).

**Zero-downtime reindex:** create `biz_v8` → bulk load snapshot (record Kafka offset) → replay CDC from offset → shadow-compare queries v7 vs v8 → swap alias → keep v7 for rollback.

**Freshness SLO:** sample N businesses/minute, compare DB version vs ES version; alert if p99 lag > 60 s.

---

### DD7. Review Trust & Safety

| Pipeline | Pros | Cons |
|---|---|---|
| Synchronous scoring before publish | Fraud never visible | Write latency; model outage blocks writes; instant feedback to attackers |
| **Async scoring after publish** | Fast writes; richer signals; delayed feedback to attackers | Fraud visible briefly |
| Human moderation only | Accurate | Doesn't scale |

**Choose async scoring + human queue for borderline cases.**

```
review.changed ──▶ Fraud scorer
   signals: account age & history · device/IP fingerprint · velocity vs business baseline
            text near-duplicates (SimHash — same as crawler content dedup)
            reviewer↔owner graph (shared IP/device, rings) · geo plausibility
   │
   ├─ low score    → recommended         (stats delta, same txn path as edits)
   ├─ high score   → not_recommended     (stats delta)
   └─ borderline   → moderation queue
```

- **Shadow filtering:** author still sees their review; others don't; stats exclude it. Reduces adversarial probing.
- **Review-bomb breaker:** velocity > k× baseline (e.g., business in the news) → freeze public reviews, show notice, batch-rescore. This is the domain's **hot-entity pattern** (cf. celebrity fan-out, spider traps).
- **Retroactive re-scoring** when models improve → batch status flips per business in one transaction each.

---

### DD8. Caching & Hot Businesses

| Cache | Key | Invalidation | Notes |
|---|---|---|---|
| CDN | immutable photo URLs | none needed | ~100% hit |
| Business page | `biz:{id}:v{version}` | CDC delete; versioned keys prevent stale re-population | Power-law traffic |
| Stats | `stats:{id}` | CDC on `business_stats` | |
| Review first page | `rev:{id}:{sort}:p1` | CDC on review change | Only hot businesses |
| Search results | `s:{S2 L13}:{norm_query}:{filters}:{15-min bucket}` | TTL 60 s | Head queries in dense cells |
| Feature store | `feat:{id}` | batch + streaming refresh | Read on every rerank |

**S2-snapped search keys:** raw lat/lng makes every key unique; snapping to ~1 km² cells lets nearby users share. Cache **ranked IDs**, recompute **distance per user** on hydration.

**Cache stampede on a hot business** (viral TikTok restaurant):

| Mitigation | How |
|---|---|
| Single-flight / request coalescing | One loader per key per node |
| Stale-while-revalidate | Serve stale while one refresh runs |
| Probabilistic early refresh | Refresh before expiry with rising probability |
| Bulkheaded DB pools | One hot key cannot exhaust connections for all |
| Local in-process L1 cache (1–5 s TTL) | Absorbs extreme single-key QPS before Redis |

---

### DD9. Meeting the Latency Budget (p99 < 200 ms)

```
Gateway + auth                     10 ms
Query understanding                10 ms
Search cache lookup                 2 ms   (hit → done in ~30 ms)
ES retrieval (1–2 shards)          50 ms
Feature fetch (500 × batch)        15 ms
Rerank                             35 ms
Hydration (cache multi-get)        10 ms
Serialization + network            20 ms
                                  ───────
                                  ~152 ms  → ~50 ms headroom for p99
```

| Technique | Effect |
|---|---|
| Geo-partitioned shards | Fewer shards per query ⇒ lower tail |
| Hedged requests to replica after p95 wait | Cuts tail from slow replicas |
| Per-stage deadlines + partial results | Never exceed budget; set `partial: true` |
| Filter caching (price, category bitsets) | ES reuses across queries |
| Reranker circuit breaker | Fallback to Stage-1 order |
| Keep docs lean; store display fields in cache | Smaller ES fetch phase |

---

### DD10. Availability & Multi-Region

| Option | Pros | Cons |
|---|---|---|
| Single write region + global read replicas | Simple consistency | ~150 ms writes for far users (acceptable for review submit) |
| **Geo-homed writes** (business's home region owns its shard) | Most writes local (people review nearby places) | Region failover procedure; routing by home region |
| Multi-master everywhere | Lowest write latency | Conflicts on uniqueness + stats → unacceptable for CP data |

**Choose geo-homed writes.** Search indexes are already regional; serve each from its region with a replica in a neighbor region for failover.

Availability tactics: reads served from replicas + caches during primary failover (Patroni, ~30 s); search continues even if Postgres is down (ES + cache are independent projections); review writes for an affected shard are rejected briefly (CP by choice).

---

### DD11. Domain Correctness Traps (pattern class)

These are Yelp's equivalent of the crawler's spider traps — **name them as a class**.

| Trap | Symptom | Fix |
|---|---|---|
| "Open now" | Closed businesses shown open after DST / overnight | Precomputed UTC intervals (DD5) |
| Antimeridian / poles | Missing results in Fiji, Aleutians | S2 covering, not lat/lng boxes |
| Cell edges | Nearby business missed across cell boundary | Cover circle with all touching cells |
| Euclidean distance on degrees | Wrong distances at high latitude | Haversine |
| Rating drift | Avg ≠ truth | Same-txn deltas + reconciliation (DD4) |
| Review bombing | Rating swings | Velocity freeze (DD7) |
| Duplicate / moved businesses | Split reviews, stale location | Entity resolution (name + geo + phone) → merge with redirect; recompute stats |
| Permanently closed | Shown in search | `status` filter in index; keep page |

---

### DD12. Efficiently Calculating & Updating Average Ratings for Search

> *"How would you efficiently calculate and update the average rating for businesses to ensure it's readily available in search results and still accurate up to the minute?"*

This is really two requirements wearing one trenchcoat, and conflating them is the trap: **"readily available"** is a *read-performance* problem, and **"accurate up to the minute"** is a *freshness* problem. They get solved by two different mechanisms, not one.

**Why you can't just compute it on the fly**

```sql
SELECT AVG(rating) FROM reviews WHERE business_id = ? AND status = 'recommended';
```
This is correct but wrong for this system: a popular business can have tens of thousands of reviews, this query would run per search *result* (up to 500 candidates per query), and at 17K QPS peak that's an unbounded scan multiplied by candidate count. It fails "readily available" immediately — O(n) per view doesn't survive this traffic.

**Step 1 — Make it O(1): maintain a precomputed aggregate, not a derived one**

A `business_stats` row (`review_count`, `rating_sum`, `histogram[5]`) is maintained as a running aggregate, updated by **delta, in the same Postgres transaction as the review write** (co-located because reviews and stats are both sharded by `business_id`):

```sql
BEGIN;
  INSERT INTO reviews (business_id, user_id, rating, ...);
  UPDATE business_stats
     SET review_count = review_count + 1,
         rating_sum   = rating_sum + :rating,
         histogram[:rating] = histogram[:rating] + 1
   WHERE business_id = :business_id;
COMMIT;
```

| Event | Delta applied |
|---|---|
| New review | count +1, sum + r |
| Edit r_old → r_new | sum + (r_new − r_old), histogram shift |
| Delete | count −1, sum − r |
| Fraud flip (recommended ↔ not_recommended) | count ±1, sum ± r |

Reading the average is now `rating_sum / review_count` — a single-row point lookup, not a scan. This is also why hot-row contention isn't a real risk here: even a viral business only takes on the order of ~0.3 review-writes/second (§4 capacity math), so the lock window on that row is negligible.

**Step 2 — Make it correct forever, not just on the happy path: reconciliation**

Deltas drift if any edge case is missed (crash mid-pipeline, a fraud-status backfill, a business merge). A **nightly batch job recomputes `SUM`/`COUNT` directly from the `reviews` table** per business and diffs it against `business_stats`. Expected drift is zero; the job's purpose is to *prove* that, and auto-repair plus alert if it isn't. This converts "trust the delta forever" into "delta for speed, batch job as the safety net" — the standard pattern for any derived running total (ledger balances, inventory counts).

**Step 3 — Get it into search without paying a per-query cost: CDC, not dual writes**

The business page reads `business_stats` directly — that's always exact, by definition, since it's the same row the transaction just updated. Search is the harder case, because the rating has to be baked into the `SearchDocument` in Elasticsearch so it can be filtered (`min_rating=4`) and scored without an out-of-band join at query time.

- The `business_stats` UPDATE is on the Postgres WAL → **Debezium → Kafka** (`stats.changed`, keyed by `business_id`).
- The indexer consumer **re-reads the full current state** of the business (not just the delta) and **re-upserts the ES doc** with `rating`, `review_count`, and a recomputed `bayesian_rating`, using `version_type=external` set to the source LSN — so if two events arrive out of order, the older one is simply rejected rather than overwriting newer data.
- A burst of edits on one business within ~1 second is **coalesced into a single re-index**, so a flurry of fraud-flip corrections doesn't trigger N redundant ES writes.

This is what makes "up to the minute" both honest and cheap: we don't promise instant search-side freshness (that would require synchronous dual writes to two different data stores per transaction, which risks partial-failure divergence), we promise **bounded** freshness — sampled and alerted on via a **freshness SLO** (compare DB version vs. ES version for N businesses/minute; alert if p99 lag > 60 s). That bound is what lets you say "up to the minute" with a number behind it instead of a vibe.

**Step 4 — Readily available ≠ raw average: what search actually ranks on**

For *ranking* (not display), the raw mean is misleading at low volumes — a single 5-star review would outrank a business with 2,000 reviews at 4.7. So the ranker uses a **Bayesian-shrunk average**, pulled toward a category prior until enough reviews accumulate:

```
bayesian = (C·m + Σr) / (C + n)        m = category mean (≈3.8), C = 20

1 review @ 5★         → (76 + 5)    / 21    ≈ 3.86
2,000 reviews @ 4.7★  → (76 + 9400) / 2020  ≈ 4.69
```

The **raw mean** (`rating_sum / review_count`) is what's *displayed* to the user; the **Bayesian average** is what's used for *sort order*. Both are precomputed and stored on the `SearchDocument`, so neither costs anything at query time.

**Putting it together — the two guarantees, side by side**

| | Business page | Search result |
|---|---|---|
| Source | `business_stats` (Postgres, same shard) | `SearchDocument.rating` (Elasticsearch) |
| Freshness | Exact — same transaction that just wrote it | Eventually consistent, p99 < 60 s (CDC lag) |
| Update mechanism | Same-txn delta | CDC re-read + external-versioned upsert |
| Cost per read | O(1) row read | O(1) field in an already-fetched document |
| Correctness backstop | Nightly SUM/COUNT reconciliation | Re-derived from `business_stats` on every CDC event, so it inherits the backstop |
| Display vs. rank | Raw mean shown | Raw mean shown; **Bayesian mean used to sort** |

**One-sentence answer, if asked cold:** *"Maintain the aggregate as a running total updated by delta inside the same transaction as the write — so it's O(1) and exact at the source — then propagate it to search via CDC with a measured freshness SLO instead of a synchronous cross-store write, and rank on a Bayesian-shrunk version of it so low-volume businesses can't game the top of the list."*

---

### DD13. Making the Synchronous Stats Update Safe Under Concurrent Writes

> *"How would you make that synchronous stats update safe if two review writes for the same business happen at nearly the same time so that `review_count` and `rating_sum` do not get out of sync?"*

**The race, concretely:** Two users review Menya Ramen within the same millisecond. Transaction A writes a 5★ review, Transaction B writes a 3★ review. If both naively compute "new total = old total + my rating" from a read taken *before* either commits, one of the updates silently clobbers the other — a classic **lost update**. The symptom is exactly the "rating drift" trap listed in DD11: `rating_sum` / `review_count` stop matching the actual rows in `reviews`.

**The broken version of this code — and why it's broken**

```python
# ANTI-PATTERN — do not do this
stats = db.query("SELECT review_count, rating_sum FROM business_stats WHERE business_id = :b")
new_count = stats.review_count + 1
new_sum   = stats.rating_sum + rating
db.execute("UPDATE business_stats SET review_count = :c, rating_sum = :s WHERE business_id = :b",
           c=new_count, s=new_sum)
```
This reads the row, computes the new *absolute* values in application memory, then overwrites the row with those absolutes. Between the `SELECT` and the `UPDATE`, another transaction can commit its own read-modify-write cycle using the same stale base value. Whichever transaction's `UPDATE` lands last **wins outright and erases the other's increment** — `review_count` ends up one too low. Isolation level doesn't save you here; this bug exists even in `SERIALIZABLE` unless you also take out a lock, because the unsafe part is doing arithmetic in the client against a value that can go stale before you write it back.

**The pattern this design actually uses (§8.4 / DD4) — and why it's safe by construction**

```sql
BEGIN;
  INSERT INTO reviews (business_id, user_id, rating, ...);
  UPDATE business_stats
     SET review_count = review_count + 1,
         rating_sum   = rating_sum + :rating
   WHERE business_id = :business_id;
COMMIT;
```

The critical difference: `review_count = review_count + 1` is **evaluated by Postgres, against the row's current value, at the moment the `UPDATE` statement actually runs** — not against a value the application read earlier. Here's why that's safe under concurrency, mechanically:

1. `UPDATE` always takes a **row-level exclusive lock** on the row it's about to modify, before applying the `SET` expression.
2. If Transaction A gets there first, it locks the `business_stats` row for Menya Ramen and holds the lock until it commits.
3. Transaction B's `UPDATE` on the *same row* **blocks** — it doesn't proceed with a stale read, it just waits.
4. The instant A commits and releases the lock, B's `UPDATE` resumes, and Postgres **re-evaluates `review_count + 1` against A's already-committed result** (this re-check-and-reapply behavior is Postgres's `EvalPlanQual` mechanism, part of how `READ COMMITTED` — the default, no special isolation level needed — handles a row that changed underneath a writer). B's `+1` is applied on top of A's `+1`, not instead of it.
5. Net effect: two concurrent `+1`s produce `+2`, in whichever commit order, every time. No lost update, no explicit `FOR UPDATE`, no retry loop in application code — the single atomic `UPDATE ... SET x = x + delta` statement *is* the concurrency control.

This is why the doc calls hot-row contention "a non-issue" (DD4): the lock window is just the duration of one tiny `UPDATE`, and even a viral business sees on the order of ~0.3 review-writes/second (§4), so writers essentially never queue behind each other in practice — but *if* they did, they'd queue safely rather than corrupt the total.

**The one place this needs an explicit lock: edits**

Creation is a pure additive delta — no prior value needs to be read. **Edits are different**, because the delta itself (`new_rating − old_rating`) depends on a value read *during* the transaction, and that read can go stale exactly the way the anti-pattern above does. That's why the edit path (DD4) adds an explicit lock on the row being read from:

```sql
BEGIN;
  SELECT rating, status FROM reviews
   WHERE business_id = :b AND review_id = :r FOR UPDATE;   -- locks the review row itself
  UPDATE reviews SET rating = :new WHERE ...;
  UPDATE business_stats SET rating_sum = rating_sum + (:new - :old) WHERE business_id = :b;
COMMIT;
```
`FOR UPDATE` on the `SELECT` locks the *specific review row*, so if the same review were somehow being edited twice at once, the second transaction blocks until the first commits, then reads the post-commit `rating` as its `:old` — the delta it computes is always relative to the true latest state, never a stale one.

**Why this can't deadlock between two reviewers of the same business:** a deadlock needs a *cycle* — two transactions each waiting on a lock the other already holds. Every write here acquires locks in the same fixed order: insert the review row (or lock the specific review row for an edit) **then** lock the `business_stats` row. Since all writers follow that same order and only ever contend for one `business_stats` row at a time, the worst case is a brief queue, never a cycle.

**Why not just use `SERIALIZABLE` isolation or an optimistic `version` column + retry loop instead?** Both would also be correct, but both add cost this workload doesn't need:
- `SERIALIZABLE` would make Postgres detect and abort conflicting transactions, requiring **application-level retry logic** for a type of conflict (blind increment) that `READ COMMITTED` already resolves for free via row locking.
- An optimistic `version` + "retry on 0 rows updated" loop (the same pattern used for owner listing edits, §6.2/§8.6) is the right tool when a **client** is round-tripping a value it read earlier (e.g., a UI showing stale hours). It's the wrong tool here because there's no client round-trip in between — the whole read-modify-write happens inside one database transaction, so a single atomic `UPDATE` already gets you exactness without ever needing a retry.

**One edge case worth naming:** the very first review for a business. `UPDATE ... WHERE business_id = :b` only works if a `business_stats` row **already exists** to be locked and updated — so the row is initialized to `(count=0, sum=0, histogram=[0,0,0,0,0])` at business-creation time, not lazily created on first review. That removes a second race (two "first reviews" both trying to `INSERT` the stats row) entirely, rather than handling it with `INSERT ... ON CONFLICT DO UPDATE`.

**One-sentence answer, if asked cold:** *"Do the increment as a single `UPDATE ... SET x = x + delta` inside the same transaction as the review write, and let Postgres's row-level locking serialize concurrent writers on that one row — never read the current total into application code and write an absolute value back, because that's where the lost-update race actually lives; only add an explicit `FOR UPDATE` when the delta itself depends on a value read earlier in the transaction, as it does on edits."*

---

### DD14. Guaranteeing One Review Per User Per Business

> *"How would you modify your system to ensure that a user can only leave one review per business?"*

Good news: nothing needs to be bolted on — the design already relies on this constraint for correctness (DD4's stats deltas assume exactly one review row per `(business_id, user_id)`), and §6.3's API already documents the `409 Conflict`. What's worth walking through is **why it's safe**, not just that it exists — because the naive version of "one review per user" is a classic TOCTOU (time-of-check-to-time-of-use) race, the same pattern class covered elsewhere for optimistic locking.

**The naive, racy version — application-level check-then-insert**

```python
# ANTI-PATTERN
existing = db.query("SELECT 1 FROM reviews WHERE business_id = :b AND user_id = :u")
if existing:
    raise Conflict("already reviewed")
db.execute("INSERT INTO reviews (business_id, user_id, rating, body) VALUES (...)")
```
This looks reasonable and passes every manual test, because a human can't double-click fast enough to trigger it. But it's broken under real concurrency: a user opens the review form in **two browser tabs**, or double-taps "Submit" on a flaky mobile connection before the UI disables the button, firing two genuinely distinct POST requests (different `Idempotency-Key`s — this is *not* the retry case §6.3 already handles). Both requests run the `SELECT`, both see "no existing review," both proceed to `INSERT`. **Two review rows exist for the same user and business.** Worse, DD4's stats math now double-counts: `review_count` and `rating_sum` both get incremented twice for what the user experienced as one submission, so the public rating is now wrong.

**The fix: push the uniqueness check into the database as a constraint, not an application `if`**

```sql
ALTER TABLE reviews ADD CONSTRAINT uq_business_user UNIQUE (business_id, user_id);
```
With this in place, the write path (§8.4) becomes:
```sql
BEGIN;
  INSERT INTO reviews (business_id, user_id, rating, body, status)
  VALUES (:b, :u, :rating, :body, 'recommended');     -- UNIQUE(business_id,user_id) enforced here, atomically
  UPDATE business_stats SET review_count = review_count + 1, rating_sum = rating_sum + :rating
   WHERE business_id = :b;
COMMIT;
```
Now replay the two-tab scenario: both `INSERT`s race, but a `UNIQUE` constraint in Postgres is backed by a B-tree index with its own internal locking — the database **guarantees** only one of the two concurrent inserts can succeed. The first commits normally. The second gets a unique-violation error *before* its `UPDATE business_stats` ever runs (the `INSERT` fails first, so the stats delta for the loser never gets applied) — the application catches that specific error and maps it to **`409 Conflict`**, exactly as documented in §6.3. There's no window where both checks can pass, because there's no separate "check" step at all — the insert attempt *is* the check, evaluated atomically by the one component (the database) that can see every concurrent writer.

**Why the shard key makes this cheap instead of a distributed problem**

This only stays a *local*, single-node constraint because reviews are sharded by **`business_id`** (DD/§8.4's shard-key decision), so every review for a given business — and therefore every potential duplicate for that business — always lands on the same Postgres shard. If reviews were instead sharded by `user_id` or `review_id`, enforcing "one review per user per business" would mean checking for duplicates **across shards**, which forces either a distributed uniqueness index, a global lock, or a cross-shard transaction — all expensive and exactly the kind of fan-out this design goes out of its way to avoid. The `UNIQUE(business_id, user_id)` constraint being enforceable as an ordinary single-node index is a direct payoff of the §8.4 shard-key choice, not a separate design decision.

**Edge case: letting a user review again after deleting their review**

A plain `UNIQUE(business_id, user_id)` constraint has a sharp edge once deletes are soft (the doc's `status` field includes `removed`, not a hard `DELETE`): if a user's old review is merely marked `removed`, the row — and the constraint — still exists, permanently blocking that user from ever reviewing that business again. That's almost certainly not the intended product behavior for a *voluntary* delete. The fix is a **partial unique index**, which enforces uniqueness only among rows that are still "live":

```sql
CREATE UNIQUE INDEX uq_review_active_business_user
  ON reviews (business_id, user_id)
  WHERE status <> 'removed';
```
Now: a user who deletes their own review (status → `removed`) is free to submit a new one — the old row no longer participates in the constraint, but it's retained for audit/history. A **fraud-driven** removal is a separate `status` value if the product wants resubmission blocked in that case specifically (e.g. `banned_from_business`), which the partial index's `WHERE` clause can exclude or include independently — this is a product decision the constraint can express either way, it just has to be made explicitly rather than falling out of a single boolean "is it unique."

**Edge case: edit vs. new review.** Because the constraint is keyed on `(business_id, user_id)` — not `(business_id, user_id, review_id)` — a user's `PUT /v1/reviews/{review_id}` edit (§6.3) is an `UPDATE` to their existing row, which never touches the unique index at all. There's no ambiguity between "editing my one review" and "creating a second review" because the schema only allows one row to exist for that pair in the first place.

**Relationship to the `Idempotency-Key`:** these are two different safeguards for two different problems, and it's worth distinguishing them explicitly if asked. The `Idempotency-Key` protects against the **same** logical request being retried (a mobile client resending after a timeout) — it returns the *original* response without re-executing anything. The `UNIQUE` constraint protects against **two different** requests (two tabs, two devices, or just a user who forgot they already reviewed) racing to create two logically distinct rows. You need both: idempotency alone wouldn't stop two-tabs-double-submit (different keys), and the unique constraint alone would still let a network retry attempt a second insert and get an unnecessary `409` instead of transparently replaying the original `201`.

**One-sentence answer, if asked cold:** *"Enforce it as a database constraint — `UNIQUE(business_id, user_id)`, or a partial unique index excluding soft-deleted rows — not an application-level check-then-insert, because the check and the insert aren't atomic in app code and a double-submit race slips through; this stays a cheap single-node constraint specifically because reviews are already sharded by `business_id`, so every potential duplicate for a business is guaranteed to land on the same shard."*

---

### DD15. Keeping the Review Insert and the Stats Update Atomic

> *"How would you structure the review write flow so the unique review insert and the `business_stats` update stay consistent if one part succeeds and the other fails?"*

The honest answer to this one is to **reject the premise rather than handle it**: the goal isn't to detect and repair the case where the insert succeeds and the stats update doesn't (or vice versa) — it's to make that case **structurally impossible**. That's the difference between *consistency by reconciliation* and *consistency by construction*, and it's worth stating explicitly, because the wrong instinct here is to reach for a saga or compensating transaction.

**Why that failure mode exists at all — the version that's vulnerable**

If the two writes are two independent statements, each committed on its own:
```sql
INSERT INTO reviews (business_id, user_id, rating, ...) VALUES (...);   -- commits
-- app crashes / connection drops / pod gets OOM-killed here --
UPDATE business_stats SET review_count = review_count + 1, rating_sum = rating_sum + :r
 WHERE business_id = :b;                                                -- never runs
```
Now a review row exists that was never counted — `review_count` and `rating_sum` permanently understate reality, with no event anywhere recording that a repair is owed. The same thing happens in reverse if these were two separate microservice calls (a Review Service write followed by a separate call to a Stats Service): a network partition after the first call succeeds but before the second one is acknowledged leaves the system in a state where only a cross-service saga with compensating logic could detect and fix it — and that machinery has to be built, tested, and kept correct forever, for a problem the database can solve for free.

**The structural fix: one ACID transaction, not two independent writes**

```sql
BEGIN;
  INSERT INTO reviews (business_id, user_id, rating, body, status)
  VALUES (:b, :u, :rating, :body, 'recommended');          -- UNIQUE(business_id,user_id), see DD14
  UPDATE business_stats
     SET review_count = review_count + 1,
         rating_sum   = rating_sum + :rating
   WHERE business_id = :b;                                  -- row-locked delta, see DD13
COMMIT;
```
A transaction's defining guarantee (the **A** in ACID) is that everything between `BEGIN` and `COMMIT` is one indivisible unit: Postgres's write-ahead log records the whole transaction, and only a transaction whose `COMMIT` has been fsynced to the WAL is considered to have happened at all. That collapses the question's "if one part succeeds and the other fails" scenario down to exactly two possible outcomes, with nothing in between:

| Outcome | What happened | What's durable |
|---|---|---|
| **Commit succeeds** | Both statements ran, both are in the WAL, fsynced | Review exists **and** stats reflect it — always together |
| **Commit never happens** (constraint violation, app crash, connection loss, process killed mid-transaction, power loss before fsync) | Automatic rollback (explicit, or implicit via crash recovery replaying only committed transactions on restart) | Neither statement's effect exists — review was never inserted, stats were never touched |

There is no third outcome where the review exists without the stats reflecting it, or the stats reflect a review that doesn't exist — the database's recovery protocol guarantees that **partial application of a transaction cannot survive a crash**, so the specific inconsistency the question asks about is ruled out at the storage-engine level, not detected-and-patched at the application level.

**Tracing it through the two concrete failure points:**
- *The `INSERT` fails* (e.g., the user already reviewed this business — DD14's unique-constraint violation): the transaction aborts right there. The `UPDATE` never executes. Stats are untouched. Client gets `409`.
- *The `UPDATE` fails* for some other reason after the `INSERT` already ran *within the same transaction* (say, a constraint or a deadlock-victim abort): Postgres rolls back the **entire transaction**, which undoes the `INSERT` too — even though it "succeeded" moments earlier, it was never committed, so from every other observer's point of view it never happened. Client gets `500` and, because the write carried an **`Idempotency-Key`** (§6.3), can safely retry the exact same request without risk of ending up with two reviews.

**What this buys beyond correctness: the client's view matches the database's view.** The service only returns `201 Created` to the client *after* `COMMIT` returns successfully — never optimistically before. So "the client was told the review was saved" and "the review and its stats delta are both durable" become the same fact, not two facts that a bug could pull apart.

**Where this boundary stops — and why that's fine, not a gap.** This transaction only spans Postgres; it says nothing about Elasticsearch, the cache, or Kafka consumers, and it shouldn't — pulling those into the same transaction would mean a slow search-indexer or a down Kafka broker could block or fail a review write, which is precisely the kind of distributed-transaction fragility this design avoids elsewhere (DD6 picked CDC over dual writes for the same reason). Those downstream systems get their consistency guarantee from a different mechanism — **at-least-once CDC delivery with external versioning and a measured freshness SLO** (DD6, DD12) — deliberately a weaker, eventually-consistent guarantee, because they don't need instant exactness; only the review row and its own business's stats counter do, and those two things live on the same shard inside one transaction specifically so they can share the strongest guarantee Postgres offers.

**Defense in depth, in case a future bug ever violates this anyway:** the nightly `SUM`/`COUNT` reconciliation job (DD4, DD12) exists precisely to catch the scenario this transaction boundary is designed to make impossible — not because the transaction is expected to fail, but because "exact" rating correctness shouldn't rest on a single mechanism never having a bug in it.

**One-sentence answer, if asked cold:** *"Put the review `INSERT` and the `business_stats` `UPDATE` inside one database transaction, so the database's own atomicity guarantee makes 'one succeeded and the other didn't' impossible rather than something to detect and repair afterward — and only acknowledge the write to the client once that transaction has actually committed, so the client's view of success and the database's durable state can never diverge."*

---

### DD16. Making Complex Queries More Efficient

> *"How can you improve search to handle complex queries more efficiently?"*

First, pin down what "complex" means here, because the answer differs by which axis of complexity is growing: a query like *"best cheap sushi or ramen, open now, 4+ stars, outdoor seating, no seafood, within 2 km of the airport"* is complex in **four independent ways** — many filter clauses, boolean logic between them (OR categories, a negation), a free-text relevance component ("best"), and a geo term that isn't even a coordinate yet ("the airport"). The baseline design (§8.2, DD1–DD3) already handles *one* filter + text + geo efficiently; this question is about what breaks as the clause count and logical complexity grow, and how to keep it fast anyway.

**1. Shrink the query before it ever reaches the search engine (query understanding does the heavy lifting)**

* The query-understanding stage (§7.1 step 3) should resolve as much of the complexity as possible into cheap, structured filters *before* Elasticsearch sees anything.
* *"the airport"* → a geocoding/entity-resolution lookup (cached — airports don't move) that turns an ambiguous phrase into a lat/lng, so the geo clause is a plain `geo_distance` filter, not a text search.
* *"sushi or ramen"* → `categories: [sushi, ramen]` as a `terms` filter (an OR expressed as a bitset union), not two separate `should` text clauses.
* *"no seafood"* → a `must_not` category/attribute exclusion filter.
* *"cheap"* → `price_tier ≤ 2`.
* What's left for actual text relevance scoring is just the genuinely unstructured residue — "best" — which is exactly what BM25 + the ranker are good at.
* **Every clause you convert from text into a structured filter moves it from O(scoring) to O(bitset AND), and that conversion is cheapest to do once, in the query-understanding layer, rather than repeatedly at the index.**

**2. Keep every filter clause in *filter context*, never *query context***

This is the single highest-leverage Elasticsearch-specific lever.

* A `bool` query's `filter` and `must_not` clauses don't contribute to scoring and are **cached as bitsets**.
* Those cached bitsets are reusable across different users' queries that happen to share a clause — e.g., thousands of different searchers all filtering `open_now=true` in the same region share that one cached bitset.
* A `must` clause, by contrast, forces per-document scoring.
* The fix for a complex multi-filter query is almost always: *audit every clause, and demote anything that isn't contributing to relevance ranking from `must`/`should` into `filter`.*
* A 10-filter query in filter context costs barely more than a 2-filter one, because bitset intersection is cheap and parallel.
* A 10-clause query where several of those clauses are accidentally in scoring context is where "complex" starts actually meaning "slow."

**3. Order filters by selectivity, most selective first**

Classic query-optimizer thinking applies directly: intersecting bitsets is cheapest when the candidate set is narrowed with the *most selective* filter first.

* **Not very selective:** `status = active` — nearly every business matches.
* **Highly selective in a sparse area:** `geo_distance ≤ 500m` in a rural region.
* **Highly selective in a dense area:** `categories = [rare_cuisine]` — can beat the radius filter in a dense city.
* Elasticsearch's own query planner handles some of this automatically.
* At this scale, it's worth maintaining lightweight **selectivity statistics per region** — approximate cardinality of each category, attribute, and price tier within each S2 region, refreshed periodically from the indexer.
* The Search Service then orders filter clauses by **observed selectivity**, not a fixed guess.
* Same idea a cost-based SQL optimizer uses — just computed for a search index instead of a table.

**4. Denormalize the hottest complex combinations at index time**

* If query logs show certain multi-clause combinations are extremely common — e.g., "open now" + "$$" + "4+ stars" + a specific category, repeated by thousands of distinct users with distinct geo centers — it's often cheaper to **precompute a composite field** than to re-intersect five bitsets per request:
  ```
  search_tags: ["cat:ramen", "price:2", "rating_tier:4plus", "open_now_bucket:evening"]
  ```
* A single `terms` match on a composite tag can replace a five-way `AND` of separate filters.
* This is the search-index analogue of a composite/covering index in a relational database — trading a bit of index size and indexing complexity for collapsing a multi-clause intersection into a single term lookup.
* Not worth doing for rare combinations — the §8.2 "Option C" precomputed-top-N browse cache already covers the simplest, highest-traffic version of this idea.
* Worth doing for the next tier of combinations: common, but not common enough to justify a full precomputed list.

**5. Keep Stage 2 (reranking) cost independent of query complexity**

* The retrieval/rerank split (DD3) already isolates the expensive part: no matter how many filter clauses Stage 1 evaluates, it still returns a bounded ~500 candidates.
* Stage 2's cost (feature fetch + model scoring) is a function of *that bound*, not of how complex the query was to construct.
* The discipline to preserve as query complexity grows: **never let a complex query increase the Stage-1 candidate cap or add per-candidate scripting.**
* A scripted filter or script-based sort (computing something per-document in a loop) is the one thing that *does* scale with query complexity in an ugly way.
* It should be avoided in favor of precomputed, indexed fields — same principle as DD5's precomputed `open_intervals` instead of a per-document time computation at query time.

**6. Cache less naively — complex queries have low cache-key reuse**

* The existing search cache (DD8) keys on `(S2 cell, normalized query, filters, time bucket)`.
* A simple query ("ramen near me") has huge key reuse across users in the same area.
* A highly complex, highly specific query has far less — fewer other users are searching for the exact same seven-filter combination at the same moment.
* So the cache hit rate drops as complexity rises, which is expected and fine — it's not a cache *failure*.
* It's a signal that **complex queries should lean on cheap filter-context bitset caching (point 2) rather than whole-result caching**.
* Bitsets for individual common sub-filters (one category, one price tier, one region) stay highly reusable even when the *combination* a given user asks for is rare.

**7. Progressive relaxation instead of a hard empty-result floor**

* A query with many `AND`-ed filters is more likely to return too few results in sparser regions — the same "sparse area" problem §8.2 already solves for radius, generalized.
* If a five-filter query returns under some threshold, **relax filters one at a time in a defined priority order** — typically: widen radius first, then loosen rating threshold, then drop a secondary category.
* This beats either returning an empty page or making the client guess which filter to remove and re-query.
* It keeps complex queries from silently becoming a UX dead end precisely because they're well-specified — the engine, not another round trip, absorbs the complexity.

**8. Watch the boolean-logic shape, not just the clause count**

* Deeply nested `bool` trees (`should` inside `must_not` inside `should`…) are more expensive to plan and execute than the same number of clauses flattened.
* They're also where correctness bugs hide — e.g., an `OR` that was meant to apply to only one sub-group leaking across the whole query.
* Query understanding should normalize the parsed intent into a **flat, canonical filter/must/should/must_not structure** before building the ES query.
* This is as much a correctness lever as a performance one.

**Putting it together, on the example query:** *"best cheap sushi or ramen, open now, 4+ stars, outdoor seating, no seafood, within 2 km of the airport"* resolves to: geocode "airport" once (cached) → `geo_distance filter`; `categories terms filter [sushi, ramen]`; `price_tier filter ≤2`; `rating filter ≥4`; `attributes filter outdoor_seating`; `attributes must_not seafood`; `open_interval date_range filter`; all in filter context, ordered by selectivity, intersected as cached bitsets within the 1–2 S2 regions the radius touches — leaving only "best" as an actual BM25 + rerank signal over whatever bounded candidate set survives the filters. The query *looks* complex to a human reading it; to the engine, it's one cheap bitset intersection plus the same Stage 1/Stage 2 pipeline every other query already uses.

**One-sentence answer, if asked cold:** *"Push as much of the complexity as possible out of free text and into structured, filter-context clauses during query understanding — geocode entities, resolve synonyms and boolean logic into terms/must_not filters — order those filters by selectivity, denormalize the hottest multi-filter combinations at index time, and make sure none of this ever increases the bounded candidate count or adds per-document scripting that Stage 2 has to pay for; the number of clauses in a query shouldn't be allowed to translate into the cost of evaluating it."*

---

### DD17. Searching by Predefined Location Names (Cities, Neighborhoods)

> *"How would you modify your system to allow searching by predefined location names such as cities (e.g., 'San Francisco') or neighborhoods (e.g., 'Mission District')? Assume you have a finite list of supported location names."*

The word **"finite"** in the question is the whole answer in disguise: an open-ended place name ("near the airport," "by that taco place I went to last year") is a hard geocoding/NLP problem. A *fixed, known-in-advance* list of cities and neighborhoods is not a search problem at all — it's a lookup table that can be fully precomputed offline, which is a much cheaper class of problem.

**1. Model each supported location as a precomputed record, not a live geocode target**

* New entity, `LocationName`: `location_id` (e.g. `sf_mission_district`), `name`, `aliases[]` (e.g. "the Mission"), `type` (city / neighborhood / state), `parent_id` (hierarchy — see point 6), `centroid`, `boundary` (GeoJSON polygon), and a precomputed `s2_cover` (the set of S2 cells covering that polygon).
* Because the list is finite and administrative boundaries essentially never change, this entire table is tiny (thousands to tens of thousands of rows globally) and can live in Postgres and be cached fully in memory — no new infrastructure class is needed.
* The `s2_cover` is computed **once per location, offline**, the same `S2RegionCoverer` operation already used per-request for circle/bbox search (§8.2) — just done ahead of time here because the input shape is fixed instead of arbitrary.

**2. Precompute business → location membership at index time (the actual efficiency unlock)**

* For every business, run a **point-in-polygon test** against the finite set of location polygons and store the result directly on the document: `SearchDocument.location_ids: ["sf_mission_district", "san_francisco", "california"]`.
* "Search in Mission District" then becomes a plain `terms` filter — `location_ids CONTAINS "sf_mission_district"` — evaluated as a **cached bitset**, exactly like any other filter-context clause (DD16, point 2). No `geo_shape` polygon math runs at query time, ever.
* This is the same move as DD16's index-time denormalization: expensive geometry is pushed to a one-time offline computation per business, not repeated per query across millions of searchers.
* Recomputing membership is cheap *because the location list is finite* — a few thousand polygon checks per business write is negligible, versus an open-ended geocoder having to search over effectively unbounded places.

**3. Reuse the existing region-routing pipeline unchanged**

* The location's precomputed `s2_cover` plugs directly into the same region-routing step that circle/bbox search already uses (§8.2, DD2) — it decides which Elasticsearch region shard(s) to query, nothing about that routing logic needs to change.
* Downstream of routing, the query is just: `location_ids` term filter ∩ any other filters (category, price, open now) ∩ BM25 text — the whole Stage 1 / Stage 2 pipeline (DD3) is untouched.
* **Net effect: a named-location search isn't a new feature inside the search engine — it's a new way to produce an S2 cover and a filter clause, both resolved before Elasticsearch is ever touched.**

**4. Resolve the user's typed text to a `location_id` before it reaches search — don't parse it at query time**

* The finite location list is small enough to serve entirely from the existing Autocomplete Service (FR6, §8.7) — add a `type: "location"` suggestion alongside the existing `category`/`business` types, backed by the same Redis prefix structure.
* The client sends the **canonical `location_id`** the user picked from the dropdown (`GET /v1/search?location_id=sf_mission_district`), not the raw string "Mission District" — so the server never has to disambiguate free text at all.
* This sidesteps the disambiguation problem by construction: if "Mission" matches both a San Francisco neighborhood and a Texas city, autocomplete shows both as distinct, hierarchy-qualified suggestions ("Mission District, San Francisco, CA" vs. "Mission, TX") with different `location_id`s — the user disambiguates once, in the UI, and the ID that reaches the backend is already unambiguous.
* This is strictly easier than DD16's "the airport" case, which needed a real geocoder for open-ended text; a finite list only needs a dictionary lookup.

**5. Handle boundary edges as an explicit product decision, not an accident**

* Strict point-in-polygon containment can exclude a business that sits right on a neighborhood's border — the named-location equivalent of DD11's "cell edges" correctness trap.
* The fix is the same shape as that trap's fix: decide deliberately whether membership tagging uses the **exact boundary** or a **small buffered version** of it (e.g., +50–100m), and apply that choice consistently in the one offline job that computes `location_ids` — not as an ad hoc runtime fudge.

**6. Keep membership fresh without re-tagging the whole index**

* **Business moves or is created:** already flows through the existing CDC pipeline (DD6) — `business.changed` → indexer re-reads full business state → the indexer also re-runs the (cheap, finite) point-in-polygon check and sets `location_ids` as part of the same document rebuild. No new pipeline needed.
* **A location's boundary itself is redrawn** (rare, administrative): triggers a scoped backfill — bounding-box-prefilter the businesses anywhere near the old or new boundary, re-run point-in-polygon only for that subset, and re-index just those documents. This is a smaller-scope cousin of DD6's full alias-swap reindex, not a new mechanism.

**7. Hierarchy for free, by denormalizing ancestors onto the same field**

* `location_ids` stores the **full ancestor chain**, not just the most specific match: a business in the Mission gets `["sf_mission_district", "san_francisco", "california"]`.
* Searching "San Francisco" then works with the exact same `terms` filter mechanism as searching "Mission District" — it's just matching a different, broader entry in the same denormalized array, no separate "is this business within any neighborhood of this city" query needed.
* **Map view tie-in (FR7):** selecting a named location from search just resolves to that `LocationName`'s stored centroid/bounding box, and the regular bbox pipeline (§8.8) takes over from there unmodified — picking a neighborhood by name and panning a map to a box are the same operation downstream of resolution.

**Putting it together:** `"coffee in the Mission"` resolves as — autocomplete picks `location_id=sf_mission_district` → its precomputed `s2_cover` routes to the 1 region shard that covers San Francisco → the query becomes `location_ids: sf_mission_district` (term filter, cached bitset) ∩ `categories: coffee` ∩ BM25 on anything left over — indistinguishable, cost-wise, from any other filtered search once resolution is done.

**One-sentence answer, if asked cold:** *"Because the location list is finite, precompute everything offline instead of solving it at query time: store each location as a polygon with a precomputed S2 cover, tag every business once (via the existing CDC pipeline) with the finite list of locations — including ancestors — that contain it, and turn 'search in X' into a plain cached-bitset term filter on that denormalized field; disambiguation is solved by autocomplete returning a canonical ID, not by the server parsing free text."*

---

## 10. CAP / PACELC Positioning

| Component | PACELC | Reasoning |
|---|---|---|
| Search index (ES) | **PA/EL** | Seconds-stale results acceptable; always serve |
| Search & page caches | **PA/EL** | Staleness bounded by TTL / CDC invalidation |
| Business listing writes | **PC/EC** | Optimistic concurrency; no lost owner edits |
| **Reviews + business_stats** | **PC/EC** | Uniqueness & exact ratings need a single serializable writer per shard; at ~70 writes/s, rejecting during partition is cheap |
| Fraud status changes | **PC/EC** | Same transactional path as reviews |
| UserReviewIndex projection | **PA/EL** | Profile lag of seconds fine |
| Photos (S3 + CDN) | **PA/EL** | Immutable objects |
| Feature store | **PA/EL** | Slightly stale features barely affect ranking |
| Region map (etcd) | **PC/EC** | Routing must be consistent; cached copy used during partition |

> *"Reads are AP everywhere — users always get results. The review write path is CP because uniqueness and rating exactness are invariants, and the write volume is tiny."*

---

## 11. Failure Modes & Mitigations

| Failure | Symptom | Detection | Mitigation |
|---|---|---|---|
| ES node/shard loss | Partial results, latency spike | Cluster health; p99 alarms | Replicas ≥ 2; partial results flagged; adaptive replica selection |
| Reranker slow/down | Budget blown | Latency SLO | Circuit breaker → Stage-1 order |
| CDC / indexer lag | New reviews/edits missing in search | Consumer lag; DB-vs-ES version sampler | Scale indexers; prioritize stats/hours topics; freshness SLO alert |
| Out-of-order index updates | Older data overwrites newer | Version sampler | External versioning |
| Rating drift | Avg ≠ truth | Nightly diff | Same-txn deltas; auto-repair + alert |
| Duplicate review on retry | Double count | Unique constraint violations | Idempotency-Key + UNIQUE(business,user) |
| Cache stampede | DB pool exhaustion | Miss spike on one key | Single-flight, SWR, L1 cache, bulkheads |
| Review bombing | Rating swings | Velocity anomaly | Freeze + batch rescore |
| Postgres primary failure | Writes fail for shard | Health checks | Automated failover; reads from replicas/cache |
| DST / tzdata change | Wrong "open now" | Spot-check job | Regenerate intervals before DST and on tzdata release |
| Hot region overload | One partition saturated | Per-partition QPS | Split region; add replicas only there |
| Photo backlog | Photos stuck processing | Queue depth | Autoscale workers; prioritize owner/cover photos |
| Scraping | Abnormal QPS from few clients | Per-client metrics | Rate limits; pagination cap; bot detection |
| Bad ranking deploy | CTR drop | A/B guardrails | Canary + auto rollback |
| Region map misconfig | Results missing for an area | Coverage test (every S2 cell maps to one region) | Validate map before publishing version; instant rollback |

---

## 12. Senior vs Staff Answer Differentiators

### Senior-level expectations

- Core services: search, business, review, media
- Pick a geo index (geohash/quadtree) and explain it
- ES for text; cache business pages; CDN for photos
- Precomputed average rating
- Reasonable APIs and capacity math

### Staff-level additions

| Topic | Staff adds |
|---|---|
| Framing | "Geo data is 6 GB — location isn't the hard part; the combined query and rating correctness are" |
| Geo | Compares all options; separates **query evaluation (BKD)** from **partitioning (S2)** |
| Search sharding | Fan-out math (435K vs 21K); hybrid geo-partitioning; tail-latency argument |
| Review sharding | business_id ⇒ local uniqueness + local stats txn; projection for profile |
| Ratings | Same-txn deltas incl. fraud flips; reconciliation; Bayesian average |
| APIs | Idempotency keys, If-Match versions, opaque cursors, partial-result flag |
| Open now | Precomputed UTC intervals; overnight/DST/tzdata |
| Index sync | CDC over dual writes; full-state re-read + external versioning; alias-swap reindex; freshness SLO |
| Trust & safety | Async scoring, shadow filtering, review-bomb freeze as hot-entity pattern |
| Latency | Explicit p99 budget; hedged requests; deadlines + partial results |
| CAP | AP reads; CP review path justified by write volume |
| Multi-region | Geo-homed writes |

**The single most important Staff differentiator:** knowing which part is hard. Spend 3 minutes on the geo index and the rest on combined search, shard-key-driven fan-out, and rating correctness — and raise "open now", review bombing, and index drift before being asked.

---

## 13. Interview Time Allocation

| Phase | Time | Focus |
|---|---|---|
| Functional & non-functional requirements | 4 min | FR1–FR8; out of scope; read-heavy; p99 |
| Capacity estimation | 4 min | QPS, storage, geo data is small, fan-out |
| Core entities + API | 5 min | Entities & shard keys; idempotency, versions, cursors |
| Data flow + high-level design | 8 min | Source of truth + CDC projections; walk FR1, FR3 |
| Deep dive: geo + search sharding + ranking | 10 min | DD1–DD3 |
| Deep dive: ratings + trust & safety | 7 min | DD4, DD7 |
| Correctness traps, CAP, failure modes | 5 min | DD5, DD11, §10, §11 |
| Remaining FRs (photos, autocomplete, map) if time | 2 min | §8.5–8.8 |

**Cut first:** autocomplete, map clustering, multi-region. **Never cut:** search sharding/fan-out, rating correctness, failure modes.

---

## 14. Quick-Reference Cheatsheet

```
CORE ENTITIES
─────────────
Business · BusinessHours · SpecialHours · Review · BusinessStats · Photo
User · OwnerResponse · OwnerClaim · (projections) SearchDocument, UserReviewIndex

KEY APIS
────────
GET  /v1/search?q&lat&lng&radius_m|bbox&filters&open_now&cursor
GET  /v1/businesses/{id}   ·  GET /v1/businesses/{id}/reviews?sort&cursor
POST /v1/businesses/{id}/reviews   (Idempotency-Key)
PUT  /v1/reviews/{id}              (If-Match)
POST /v1/photos/upload-intent → presigned URL
GET  /v1/autocomplete?prefix&lat&lng

KEY ALGORITHMS
──────────────
Geo query:     ES geo_point (BKD)         Partitioning: S2 cells → balanced regions
Ranking:       ES retrieval (500) → reranker (20) → breaker fallback
Rank rating:   (C·m + Σr) / (C + n)
Pagination:    keyset / search_after, opaque cursor
Open now:      UTC open_intervals for 7 days, date_range filter
Index sync:    CDC → Kafka (key=business_id) → re-read state → ES external version
Ratings:       same-txn deltas + nightly reconciliation

KEY NUMBERS (illustrative)
──────────────────────────
Search 5.8K avg / 17K peak QPS · pages 11.6K / 35K · reviews ~70 writes/s peak
Geo data ~6.4 GB · search index ~1 TB (25 shards) · photos 6 TB/day
Fan-out: 435K (doc-hash) vs ~21K (geo-partitioned) shard-queries/s

CAP
───
AP/EL: search, caches, projections, photos, feature store
PC/EC: reviews + stats, listing writes, fraud status, region map

STAFF SIGNALS
─────────────
✓ Geo isn't the hard part   ✓ Shard key decides fan-out
✓ business_id review shard → local uniqueness + stats txn
✓ Exact ratings incl. fraud flips + reconciliation   ✓ Bayesian average
✓ CDC over dual writes; external versioning; alias swap
✓ Open-now DST/overnight   ✓ Review bombing = hot-entity pattern
✓ Explicit p99 budget, hedging, partial results   ✓ Idempotency keys in API
```

---

## 15. Related Guides

| Guide | Relationship |
|---|---|
| `proximity_service_system_design.md` | Geo index internals; this guide only decides |
| `sharding_strategies_system_design.md` | Shard key → fan-out; logical shards; rebalancing |
| `cap_pacelc_theorem_system_design.md` | Per-component PACELC framing |
| `apache_kafka_system_design.md` | CDC topics, per-key ordering, consumer lag, `acks` + `min.insync.replicas` |
| `file_upload_malware_scanning_system_design.md` | Presigned upload and scanning for photos |
| `circuit_breaker_pattern_system_design.md` | Reranker fallback |
| `distributed_unique_id_generation_system_design.md` | Snowflake IDs for business/review/photo |
| `news_feed_system_design.md` | Hot-entity pattern ↔ review bombing |
| `web_crawler_system_design.md` | Spider-trap pattern class ↔ domain traps; SimHash reuse |
| `notification_system_design.md` | Owner notifications for new reviews |
| `leaderboard_system_design.md` | "Top-rated in city" lists; same shard-key/fan-out tradeoff |

---

*Guide compiled for Senior & Staff-level system design interview preparation.*
*Topics: Yelp, Local Search, Geospatial Indexing, Ranking, Reviews, Rating Aggregation, CDC, Trust & Safety.*
