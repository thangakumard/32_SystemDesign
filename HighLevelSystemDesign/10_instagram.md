# Instagram — System Design Interview Guide
### Senior & Staff Engineer Level

> All numbers in this guide are **illustrative** interview-grade estimates, not Instagram's published figures. Decimal (SI) units throughout.

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
9. [Deep Dive 1: Creating Posts with Photos/Videos](#9-deep-dive-1-creating-posts-with-photosvideos)
10. [Deep Dive 2: Following Users](#10-deep-dive-2-following-users)
11. [Deep Dive 3: Chronological Feed](#11-deep-dive-3-chronological-feed)
12. [Deep Dive 4: Scaling Feeds for Users Who Follow Thousands](#12-deep-dive-4-scaling-feeds-for-users-who-follow-thousands)
13. [Deep Dive 5: Uploading Large Media (up to 4 GB)](#13-deep-dive-5-uploading-large-media-up-to-4-gb)
14. [Deep Dive 6: Fast Global Media Delivery](#14-deep-dive-6-fast-global-media-delivery)
15. [CAP / PACELC Positioning](#15-cap--pacelc-positioning)
16. [Failure Modes & Mitigations](#16-failure-modes--mitigations)
17. [Scalability & Sharding](#17-scalability--sharding)
18. [Senior vs Staff Answer Differentiators](#18-senior-vs-staff-answer-differentiators)
19. [Interview Time Allocation](#19-interview-time-allocation)
20. [Quick-Reference Cheatsheet](#20-quick-reference-cheatsheet)
21. [Related Guides](#21-related-guides)

---

## 1. Problem Statement & Scope

Instagram is a media-first social network: users publish posts containing photos and/or videos, follow other users, and consume a feed of posts from the accounts they follow.

### In scope
- Post creation with photos and videos (up to 4 GB per video)
- Follow / unfollow
- **Chronological** home feed of followed accounts
- Global, low-latency media delivery

### Out of scope (call out, then park)
- Ranked/ML feed, Explore, Reels recommendations
- Stories, DMs, comments, likes (mention counters briefly only)
- Search, ads, content moderation internals (mention as a pipeline hook)

### What the interviewer is really testing

| Signal | What it looks like |
|---|---|
| Read-heavy fan-out design | Push vs pull vs hybrid, with the celebrity problem raised **unprompted** |
| Large-object handling | Control plane (metadata) separated from data plane (bytes); app servers never touch media bytes |
| Global delivery | CDN tiers, immutable URLs, adaptive bitrate — not just "put a CDN in front" |
| Operational ownership | Failure modes, backfill, orphan cleanup, hot keys, per-component CAP |

**Anchor constraint (drives everything downstream):** *Media bytes and metadata take completely different paths.* Bytes go client → object store → CDN; metadata goes client → API → databases/caches. Every Staff-level decision below falls out of keeping those planes separate.

---

## 2. Functional Requirements

| # | Requirement | Notes |
|---|---|---|
| FR1 | Users create posts with one or more photos/videos + caption | Carousel up to ~10 items; video ≤ 4 GB |
| FR2 | Users follow / unfollow other users | Public accounts: instant; private accounts: follow request → approve |
| FR3 | Users see a chronological feed of posts from accounts they follow | Newest first, cursor-paginated, infinite scroll |
| FR4 | Users view a profile's posts | Author timeline ("profile grid") |
| FR5 | Media plays/renders on any device and network | Multiple renditions, adaptive bitrate for video |

---

## 3. Non-Functional Requirements

| Requirement | Target | Why it matters |
|---|---|---|
| Scale | 2B registered users, 500M DAU | Drives sharding and fan-out cost |
| Feed latency | p99 < 500 ms for first page (metadata); p50 < 200 ms | Feed open is the core engagement moment |
| Media time-to-first-render | Image < 1 s, video first frame < 1–2 s globally | Users scroll away from blank tiles |
| Availability | 99.99% for feed reads; 99.9% for post creation | Reads matter more than writes for perceived uptime |
| Consistency | Eventual for feed (seconds acceptable); **read-your-writes** for own posts | A user must see their own post immediately |
| Durability | 11 nines for media; zero post loss after acknowledgment | Lost photos are unrecoverable trust loss |
| Upload robustness | Resumable uploads; survive network drops & app kills | Mobile networks are lossy; 4 GB uploads take minutes–hours |
| Privacy | Private accounts, blocks enforced on **every** read | Correctness-critical — the one area that is not "eventually fine" |

**Read:write ratio:** roughly 50:1 at the metadata layer, and far higher at the media layer (one upload → thousands of views).

---

## 4. Capacity Estimation

```
Assumptions (illustrative):
  DAU                        = 500M
  New posts/day              = 100M   (90% photo, 10% video)
  Feed opens/user/day        = 10
  Avg follows per user       = 200    (=> avg followers also 200)
  Avg photo (post-client-resize) = 2 MB original
  Avg video original         = 50 MB  (max 4 GB)

Write QPS (posts):
  100M / 86,400              ≈ 1,160 posts/s   (peak ~3× ≈ 3.5K/s)

Read QPS (feed):
  500M × 10 / 86,400         ≈ 58K feed reads/s (peak ~150K/s)

Fan-out-on-write (push) volume:
  100M posts × 200 followers ≈ 20B timeline inserts/day
                             ≈ 231K inserts/s  (internal, not client-facing)

Media ingest (originals):
  Photos: 90M × 2 MB         = 180 TB/day
  Videos: 10M × 50 MB        = 500 TB/day
  Total originals            ≈ 680 TB/day   ≈ 63 Gbps sustained upload
  + renditions (~1×)         ≈ ~1.4 PB/day  ≈ ~500 PB/year

Media egress:
  5B feed opens × ~10 media items × ~300 KB avg delivered
                             ≈ 15 PB/day    ≈ 1.4 Tbps avg (peak ~4 Tbps)
  => only feasible via CDN; origin should serve < 5–10% of bytes

Precomputed timeline cache:
  500M active users × 500 entries × 16 B (post_id + ts)
                             ≈ 4 TB  (×3 replicas ≈ 12 TB Redis)

Follow graph:
  2B users × 200 follows × 2 directions × ~64 B
                             ≈ 51 TB  (sharded wide-column)
```

**Key insights**
1. **Media dominates storage and bandwidth** by 3+ orders of magnitude over metadata. Media is an object-store + CDN problem, not a database problem.
2. **Feed reads (58K/s) are the hot path**, but fan-out writes (231K/s internal) are the hidden cost — note the difference between client-facing and internal fan-out QPS.
3. **4 TB of timelines fits in memory.** Precomputing feeds is affordable; that's what makes push viable.

---

## 5. Core Entities

```
User
  user_id (PK, 64-bit), username, display_name, avatar_media_id,
  is_private, follower_count, following_count, created_at

Post
  post_id (PK, 64-bit time-sortable Snowflake), author_id,
  caption, media_ids[] (ordered), status (DRAFT|PROCESSING|PUBLISHED|DELETED),
  visibility, created_at, location (optional)

Media
  media_id (PK), owner_id, type (PHOTO|VIDEO), status
    (UPLOADING|UPLOADED|PROCESSING|READY|FAILED),
  original_key (object store), renditions{ "1080w":key, "640w":key, ...,
  "hls_master":key }, width, height, duration, checksum, size_bytes

UploadSession
  upload_id (PK), media_id, user_id, object_key, s3_multipart_id,
  part_size, total_parts, completed_parts (bitmap), expires_at

Follow (edge — stored twice, one per direction)
  following:  (follower_id, followee_id, created_at)   partition: follower_id
  followers:  (followee_id, follower_id, created_at)   partition: followee_id
  status: ACTIVE | PENDING (private account request)

Timeline (cache, derived — not source of truth)
  key: timeline:{user_id}  →  [(post_id, ts)] newest first, capped ~500

AuthorPosts (index; source for profile grid & celebrity pull)
  key: (author_id) → [post_id] clustering desc
```

Why Snowflake IDs for `post_id`? They are **time-sortable**, so "chronological" = "sort by ID", and cursor pagination becomes `post_id < cursor`. See *Distributed Unique ID Generation* guide.

---

## 6. API / System Interface

All endpoints authenticated (OAuth2 bearer); `user_id` comes from the token, never the body.

### Media upload (control plane only — bytes never pass through the API)

```http
POST /v1/media/uploads
{ "type": "VIDEO", "size_bytes": 3800000000, "content_type": "video/mp4",
  "sha256": "…", "client_upload_key": "uuid-for-idempotency" }
→ 201 { "media_id": "m_123", "upload_id": "u_456",
        "part_size": 16000000, "total_parts": 238,
        "part_urls_endpoint": "/v1/media/uploads/u_456/parts" }

GET  /v1/media/uploads/{upload_id}/parts?from=1&count=50
→ 200 { "parts": [{ "n":1, "url":"https://…presigned…", "expires_at":… }, …] }

GET  /v1/media/uploads/{upload_id}            # resume: which parts are done?
→ 200 { "completed_parts": [1,2,3,…], "status": "UPLOADING" }

POST /v1/media/uploads/{upload_id}/complete
{ "parts": [{ "n":1, "etag":"…" }, …] }
→ 202 { "media_id": "m_123", "status": "PROCESSING" }
```

Small photos use the same flow with `total_parts = 1` (single presigned PUT).

### Posts

```http
POST /v1/posts
Idempotency-Key: 7c1e…
{ "media_ids": ["m_123","m_124"], "caption": "Sunset 🌅", "location_id": null }
→ 202 { "post_id": "p_789", "status": "PROCESSING" }   # → PUBLISHED when media READY

GET    /v1/posts/{post_id}
DELETE /v1/posts/{post_id}
GET    /v1/users/{user_id}/posts?cursor=<post_id>&limit=24     # profile grid
```

### Follow graph

```http
POST   /v1/users/{target_id}/follow      → 200 { "status": "ACTIVE" | "PENDING" }
DELETE /v1/users/{target_id}/follow      → 204
POST   /v1/follow-requests/{requester_id}/approve
GET    /v1/users/{user_id}/followers?cursor=…&limit=50
GET    /v1/users/{user_id}/following?cursor=…&limit=50
```

### Feed

```http
GET /v1/feed?cursor=<post_id>&limit=20
→ 200 {
  "items": [{
     "post_id": "p_789", "author": { "id":…, "username":…, "avatar_url":… },
     "caption": "…", "created_at": "…",
     "media": [{ "type":"PHOTO", "placeholder":"<blurhash>",
                 "urls": { "320w":"https://cdn…/…", "640w":…, "1080w":… },
                 "width":1080, "height":1350 },
               { "type":"VIDEO", "hls":"https://cdn…/master.m3u8",
                 "poster":"https://cdn…/poster.jpg", "duration_ms":… }]
  }],
  "next_cursor": "p_701"
}
```

**Design choices to defend**
- **Cursor, not offset**, pagination: offsets break when new posts arrive while scrolling (duplicates / skips).
- **Feed returns hydrated items** (not just IDs) to avoid N+1 round trips from mobile clients on high-latency networks.
- **`Idempotency-Key` on post creation**: mobile retries after a timeout must not create duplicate posts.
- **Write endpoints return 202** where work is async (media processing), with status polling or a push notification on completion.

---

## 7. Data Flow

### 7.1 Create a post (FR1)

```
1. Client resizes/compresses photo locally (or prepares video)
2. POST /media/uploads           → API creates Media(UPLOADING) + UploadSession
3. Client PUTs bytes directly    → Object store (presigned URLs, multipart, parallel)
4. POST /uploads/{id}/complete   → API calls CompleteMultipartUpload
                                   Media → UPLOADED; event "media.uploaded" → Kafka
5. Media Processing workers      → validate, scan, strip EXIF GPS, make renditions /
                                   transcode HLS ladder; Media → READY
6. POST /posts (any time after step 2, idempotent)
                                 → Post(PROCESSING) written to Post DB
7. When all media READY          → Post → PUBLISHED; event "post.published" → Kafka
8. Fan-out service               → pushes post_id into followers' timelines (Deep Dive 3)
```

### 7.2 Follow (FR2)

```
1. POST /users/{B}/follow by A
2. Graph service: check block list; if B is private → write PENDING edge, notify B
   else write ACTIVE edge to `following` (source of truth)
3. CDC / Kafka "follow.created"  → derive `followers` row, increment counters
4. Timeline backfill worker      → merge B's last N posts into A's timeline
```

### 7.3 Read the feed (FR3)

```
1. GET /feed?cursor=X
2. Feed service reads timeline:{A} from Redis (push-delivered post_ids)
3. Merge in recent post_ids from followed celebrities (pull) — Deep Dive 4
4. Merge in A's own recent posts (read-your-writes)
5. Take top `limit` by post_id < X
6. Hydrate: Post cache → User cache → media URL construction
7. Filter: deleted posts, blocks, revoked private-account access
8. Return items + next_cursor
```

### 7.4 View media (FR5)

```
Client → GeoDNS/Anycast → nearest CDN edge
   hit  → served from edge
   miss → regional shield cache → (miss) → origin object store (nearest replica)
Video: player fetches master.m3u8 → picks rendition → fetches 2–6 s segments,
       switching bitrate with measured throughput
```

---

## 8. High-Level Design

```
                                   ┌──────────────────────────────┐
                                   │       Mobile / Web Client    │
                                   └──────┬────────────────┬──────┘
                       metadata (HTTPS)   │                │  media bytes (upload & download)
                                          │                │
                        ┌─────────────────▼───┐     ┌──────▼──────────────────────────┐
                        │  API Gateway / LB   │     │  CDN (edge POPs → regional       │
                        │  auth, rate limit   │     │  shield) — downloads             │
                        └──┬──────┬──────┬────┘     │  Upload edge / accel — uploads   │
                           │      │      │          └──────┬───────────────▲──────────┘
             ┌─────────────┘      │      └───────────┐     │ presigned PUT │ cache fill
             │                    │                  │     │               │
     ┌───────▼───────┐   ┌────────▼───────┐  ┌───────▼─────▼───────────────┴────────┐
     │ Post Service  │   │ Graph Service  │  │  Media Service (control plane)       │
     │ create/delete │   │ follow/unfollow│  │  upload sessions, presign, status    │
     └───┬───────┬───┘   └───┬────────┬───┘  └───────┬───────────────────┬──────────┘
         │       │           │        │              │                   │
  ┌──────▼──┐    │    ┌──────▼─────┐  │      ┌───────▼──────┐   ┌────────▼─────────────┐
  │ Post DB │    │    │ Graph DB   │  │      │ Media Meta DB│   │ Object Store (S3)    │
  │Cassandra│    │    │ following /│  │      │ + UploadSess │   │ originals + renditions│
  │by author│    │    │ followers  │  │      └──────────────┘   │ multi-region         │
  └─────────┘    │    └────────────┘  │                         └────────▲─────────────┘
                 │                    │                                  │ read/write
                 ▼                    ▼                                  │
        ┌────────────────────────────────────────────────┐     ┌────────┴─────────────┐
        │                 Kafka (event bus)               │────▶│ Media Processing     │
        │ post.published · follow.* · media.uploaded      │◀────│ image resize, video  │
        └───────┬──────────────────────┬──────────────────┘     │ transcode (HLS/DASH),│
                │                      │                        │ thumbnails, scanning │
       ┌────────▼────────┐   ┌─────────▼────────┐               └──────────────────────┘
       │ Fan-out Workers │   │ Backfill /       │
       │ push post_id to │   │ Counter Workers  │
       │ active followers│   └─────────┬────────┘
       └────────┬────────┘             │
                ▼                      ▼
        ┌────────────────────────────────────────┐
        │ Timeline Cache (Redis Cluster)          │◀──── read ────┐
        │ timeline:{user} ZSET, capped ~500       │               │
        │ author_recent:{celebrity} lists         │       ┌───────┴────────┐
        └────────────────────────────────────────┘       │  Feed Service  │◀── GET /feed
                                                          │ merge + hydrate│    (via Gateway)
        ┌────────────────────────────────────────┐       └───────┬────────┘
        │ Hydration caches (Memcached/Redis):     │◀──────────────┘
        │ post:{id}, user:{id}, blocklist:{id}    │
        └────────────────────────────────────────┘
```

### Component responsibilities & why

| Component | Responsibility | Why this choice |
|---|---|---|
| API Gateway | AuthN, rate limiting, routing, request size limits | Keeps services stateless; rejects abuse early |
| Post Service | Validates post, writes metadata, publishes events | Owns post lifecycle state machine |
| Media Service | Creates upload sessions, presigns URLs, tracks status | **Control plane only** — never proxies bytes |
| Object Store (S3/GCS) | Durable media bytes, multipart uploads | 11-nines durability, effectively infinite scale, cheap |
| Media Processing | Resize, transcode, thumbnail, EXIF strip, safety scan | CPU/GPU-heavy, bursty → separate autoscaled fleet off Kafka |
| Graph Service + DB | Follow edges in both directions, counts, follow requests | High write volume, simple access patterns → wide-column |
| Kafka | Durable decoupling of publish → fan-out, follow → backfill | Replay, back-pressure, absorbs celebrity bursts (see *Kafka* guide) |
| Fan-out Workers | Push post IDs into followers' timelines | Moves feed cost from read path to write path |
| Timeline Cache | Precomputed per-user post-ID lists | Makes feed reads O(1) w.r.t. following count |
| Feed Service | Merge push + pull + own posts, hydrate, filter | Single place to enforce privacy on read |
| CDN | Edge-cache media globally | 1.4 Tbps egress cannot come from origin |

### Functional requirement coverage

| FR | Path through the diagram |
|---|---|
| FR1 create post | Client → Media Service (session) → S3 (bytes) → Processing → Post Service → Kafka → Fan-out |
| FR2 follow | Client → Graph Service → Graph DB → Kafka → Backfill + Counters |
| FR3 feed | Client → Feed Service → Timeline Cache (+ celebrity pull) → Hydration caches → response |
| FR4 profile | Client → Post Service → Post DB partition `author_id` (cached) |
| FR5 media render | Client → CDN edge → shield → S3 |

---

## 9. Deep Dive 1: Creating Posts with Photos/Videos

### 9.1 The core decision: how do bytes and metadata travel?

| Option | How | Pros | Cons |
|---|---|---|---|
| **A. Single multipart/form request to API** | Client POSTs caption + files to app server, which writes to S3 | Simplest; atomic from client view | App servers become a bandwidth bottleneck (63 Gbps ingest); long-lived connections tie up threads; no resume; 4 GB through a gateway is a non-starter |
| **B. API streams to S3 (proxy)** | App server pipes the stream | Server can inspect bytes inline | Still pays bandwidth + connection time on app tier; failure mid-stream loses everything |
| **C. Presigned direct-to-object-store, then create post** ✅ | Client gets presigned URL(s), uploads directly, then calls `POST /posts` with `media_ids` | App tier handles only tiny JSON; S3 absorbs bandwidth; resumable; scales independently | Two-step flow → must handle orphans (uploaded media never attached) and post/media status coordination |
| **D. Post-first, media later** | Create post shell, attach media as it uploads | Good UX (post appears "processing") | Same as C plus more partial states visible to followers unless gated |

**Choice: C** (with the post shell from D as a UX refinement — the post exists but is not fanned out until `PUBLISHED`).

### 9.2 Post lifecycle state machine

```
              create (idempotent)
   ┌──────┐  ─────────────────────▶ ┌────────────┐   all media READY   ┌───────────┐
   │ none │                         │ PROCESSING │ ──────────────────▶ │ PUBLISHED │
   └──────┘                         └─────┬──────┘                     └─────┬─────┘
                                          │ any media FAILED / timeout       │ delete
                                          ▼                                  ▼
                                    ┌──────────┐                       ┌──────────┐
                                    │  FAILED  │                       │ DELETED  │ (tombstone)
                                    └──────────┘                       └──────────┘
```

- Transitions are **conditional writes** (`UPDATE … IF status = 'PROCESSING'`, i.e. compare-and-set) so duplicate `media.ready` events can't double-publish.
- `post.published` is emitted via a **transactional outbox** (or CDC on the Post table), never a "write DB then publish Kafka" dual write — otherwise a crash between the two leaves a post that's in the DB but never fanned out.
- Only `PUBLISHED` posts are fanned out or returned in feeds.

### 9.3 Idempotency

```
POST /posts  Idempotency-Key: K
  1. SETNX idem:{user}:{K} → "in_progress" (TTL 24h)
     - exists with result → return stored response (same post_id)
     - exists in_progress → 409 / retry-after
  2. Generate post_id (Snowflake), write Post
  3. Store response under idem key
```

Media creation is idempotent on `client_upload_key` the same way, so an app restart that re-calls "create upload" resumes the existing session instead of starting a second 4 GB upload.

### 9.4 Photo processing pipeline

```
media.uploaded ─▶ Validate (magic bytes, dimensions, decompression-bomb guard)
              ─▶ Safety scan (CSAM hash match, malware for non-image payloads)
              ─▶ Strip EXIF (GPS!) — privacy-critical; keep orientation
              ─▶ Generate renditions: 150w, 320w, 640w, 1080w
                  formats: AVIF + WebP + JPEG fallback
              ─▶ Compute blurhash / tiny LQIP placeholder (~30 bytes)
              ─▶ Write renditions to S3 under content-addressed keys
              ─▶ Media → READY (conditional write) ─▶ media.ready event
```

**Client-side resize before upload** (e.g., cap long edge at 1080–1440 px) cuts a 12 MP / 5–8 MB phone photo to ~1–2 MB. This is the single biggest cost lever on ingest and is why the capacity math assumes 2 MB.

### 9.5 Storage for posts

| Option | Pros | Cons |
|---|---|---|
| Sharded PostgreSQL (shard by `author_id`) | Rich queries, transactions, mature ops | Manual resharding; write scaling needs care |
| **Cassandra / DynamoDB, partition `author_id`, cluster `post_id DESC`** ✅ | Linear write scaling; profile grid = single-partition range scan; tunable consistency | No joins; secondary access patterns need denormalized tables |
| Document DB (MongoDB) | Flexible schema for carousels | Weaker multi-region story at this scale |

Primary lookup by `post_id` (for hydration) is served by the **post cache**; a `post_by_id` table (partition `post_id`) backs cache misses. Both are written from the same outbox event.

---

## 10. Deep Dive 2: Following Users

### 10.1 Access patterns
1. "Who does A follow?" → feed merge, following list (partition by follower)
2. "Who follows B?" → fan-out on publish, followers list (partition by followee)
3. "Does A follow B?" → private-account authorization, follow button state
4. Counts for profile header

One table cannot serve (1) and (2) efficiently, so the edge is **stored twice**.

### 10.2 Storage options

| Option | Pros | Cons | Verdict |
|---|---|---|---|
| Relational `follows(follower, followee)` with two indexes | Simple; ACID dual-direction | Secondary index on a sharded table = scatter-gather across shards | Fine to ~100M edges; not 400B |
| **Wide-column, two tables (`following`, `followers`)** ✅ | Each query is a single-partition scan; linear scale | Two writes per follow → must keep them consistent | Chosen |
| Graph DB (Neo4j) | Multi-hop queries (friends-of-friends) | Hard to shard at 400B edges; we only need 1-hop | Overkill |
| Purpose-built graph store over sharded SQL + cache (TAO-style) | Massive read scale, assoc lists per object | Large build/ops investment | What a mega-scale org converges to; mention as evolution |

### 10.3 Keeping both directions consistent

| Approach | Pros | Cons |
|---|---|---|
| Client-side dual write | Simple | Partial failure leaves edges asymmetric forever |
| Cassandra logged batch | Atomic eventual (both or neither, eventually) | No isolation; batchlog overhead on a hot path |
| **Single source of truth (`following`) + CDC → derive `followers`** ✅ | One authoritative write; derivation is idempotent & replayable | `followers` lags by seconds; reconciliation job needed for drift |

Fan-out tolerates a few seconds' lag in `followers` (worst case: a brand-new follower misses one post, which backfill catches).

### 10.4 Follow side effects (async, via `follow.created`)

| Side effect | How |
|---|---|
| Counters | Sharded counters / Redis `INCR`, periodically reconciled from edge counts; **AP** — off-by-a-few is fine |
| Timeline backfill | Fetch B's last ~20 posts, `ZADD` into `timeline:A`; idempotent because ZSET members are unique |
| Celebrity handling | If B is a celebrity (pull-mode author), skip backfill — B is merged at read time anyway |
| Notification | "B started following you" via the *Notification System* |

### 10.5 Unfollow, block, private accounts

- **Unfollow:** delete edge; async remove B's posts from `timeline:A` (`ZREM` by member scan, or simply let the read-time filter handle it and age out).
- **Block:** must be **immediate and strongly consistent** — write both directions, and the Feed Service checks a block-list cache at hydration so a stale timeline entry never leaks.
- **Private accounts:** follow creates a `PENDING` edge; only `ACTIVE` edges participate in fan-out. If a private account later removes a follower, the read-time authorization check (not the timeline) is the guarantee.

> **Staff signal:** Timelines are a *performance cache*, never an *authorization* source. All privacy decisions happen at read/hydration time against the graph's source of truth.

### 10.6 Abuse limits
Follow rate limits (per minute/day) and a cap on total following (Instagram reportedly caps around 7,500) bound the "follows thousands" problem — which matters for Deep Dive 4.

---

## 11. Deep Dive 3: Chronological Feed

### 11.1 The three canonical strategies

| Strategy | Write cost | Read cost | Pros | Cons |
|---|---|---|---|---|
| **Pull (fan-out-on-read)** | O(1) — write to author's list | O(F) — query every followed author, k-way merge | No write amplification; always fresh; simple deletes/unfollows | Read latency grows with following count; 58K/s × 200 authors = ~12M partition reads/s |
| **Push (fan-out-on-write)** | O(followers) per post | O(1) — read one precomputed list | Fast, predictable reads | Celebrity with 600M followers = 600M writes per post; wasted work for inactive followers |
| **Hybrid** ✅ | Push for normal authors; pull for celebrities | O(1) + O(celebrities followed) | Bounded write amplification **and** bounded read cost | Two code paths; merge logic; threshold tuning |

### 11.2 Hybrid design

```
On publish (author X, follower count N):
  if N ≤ CELEB_THRESHOLD (e.g. 100K, illustrative) and X not flagged celebrity:
      PUSH: for each ACTIVE follower f who was active in last 30 days:
                ZADD timeline:f  score=ts_ms  member=post_id
                ZREMRANGEBYRANK timeline:f 0 -501      # cap at 500
  else:
      PULL: LPUSH author_recent:X post_id; LTRIM to 200
            (no per-follower writes)
  Always: append to AuthorPosts (Post DB) for profile & cold rebuild
```

```
On GET /feed (user A, cursor C):
  1. pushed  = ZREVRANGEBYSCORE timeline:A  (< C)  LIMIT 2×page
  2. celebs  = A's followed celebrities (cached set, refreshed on follow events)
     pulled  = parallel LRANGE author_recent:{celeb} for each, filter < C
  3. own     = A's own posts in the window (read-your-writes)
  4. merged  = k-way merge by post_id desc; dedupe; take `limit`
  5. hydrate + filter (deleted, blocked, privacy) → backfill if page short
  6. next_cursor = last returned post_id
```

### 11.3 Redis data-structure trap (Staff-level correctness)

Redis ZSET scores are **IEEE-754 doubles (53-bit mantissa)**. A 64-bit Snowflake `post_id` **cannot be stored exactly as a score** — adjacent IDs collapse, breaking ordering and cursors.

| Option | Verdict |
|---|---|
| score = post_id | ❌ precision loss above 2^53 |
| score = millisecond timestamp (41 bits — fits), member = post_id | ✅ ties within the same ms ordered lexicographically by member; cursor = (ts, post_id) |
| Redis LIST with LPUSH + LTRIM | ✅ cheap, but insertion order ≠ creation order under fan-out lag (delayed post inserted on top) |

### 11.4 Inactive users & cold start
- **Skip fan-out to users inactive > 30 days** (huge savings — a large share of followers are dormant).
- On their next login, the timeline cache is missing → **rebuild by pull**: read the following list, fetch each author's recent posts from `AuthorPosts`, merge, write back to Redis. Slow once (~hundreds of ms), then fast.
- Timeline cache entries carry a TTL (e.g., 30 days) so dormant users don't occupy memory.

### 11.5 Consistency guarantees

| Scenario | Guarantee & mechanism |
|---|---|
| I post, then open my feed | **Read-your-writes**: Feed Service merges the user's own recent posts directly from the Post DB/cache, independent of fan-out lag |
| Followee posts | Eventual, typically < 5 s (fan-out queue lag SLO) |
| Post deleted | Hidden immediately via tombstone check at hydration; timeline entries age out lazily |
| Unfollow / block | Unfollow: filtered at read; Block: enforced at read (CP) |
| Deep scroll beyond 500 cached items | Fall back to pull from `AuthorPosts` for older pages |

### 11.6 Pagination stability
Cursor = last seen `(ts, post_id)`. New posts arriving at the top never shift older pages — unlike offset pagination. "New posts" pill = a separate lightweight query `> first_seen_cursor`.

---

## 12. Deep Dive 4: Scaling Feeds for Users Who Follow Thousands

### 12.1 Reframe the problem precisely

There are two different hot spots that interviewers conflate:
- **High-*follower* authors (celebrities)** → write amplification on push. Solved by pull for those authors.
- **High-*following* readers (follow 5,000 accounts)** → read amplification on pull. *This* is the question.

With pure pull, a user following 5,000 accounts needs 5,000 partition reads + a 5,000-way merge per feed open — easily seconds. The design goal: **make read cost independent of following count.**

### 12.2 Why hybrid already handles most of it

Under the hybrid model, the 5,000 followed accounts split into:
- ~4,700 normal authors → **already pushed** into `timeline:A`. Read cost: one ZSET range read, regardless of whether it came from 10 or 4,700 authors.
- ~300 celebrities → **pulled at read time**. This residual is the real problem.

### 12.3 Techniques to bound the celebrity pull

| Technique | How it works | Pros | Cons |
|---|---|---|---|
| **1. "Recently active" filter** | Keep `celeb_last_post_ts:{id}` in a small hot cache. Only pull celebrities whose last post is newer than the oldest item in the current page window | Typically reduces 300 → a handful (most celebrities didn't post in the window) | One extra MGET of 300 timestamps (~1 ms, single batched call) |
| **2. Batched, shard-aware fetch** | Group `author_recent:*` keys by Redis slot; one pipelined call per node, all in parallel | Turns 300 round trips into ~10 parallel ones | Tail latency = slowest shard → hedged requests |
| **3. Bounded k-way merge** | Min-heap of size k, pull only `limit` items; each list already sorted | O(limit · log k), microseconds | — |
| **4. Per-reader push override** | For "heavy readers" (high activity, follow many celebs), *do* push celebrity posts to them — decision per **edge**, not per author | Reader cost back to O(1) for the most demanding users | More write amplification; needs an "is heavy reader" list consulted in fan-out |
| **5. Precomputed celebrity merge cache** | Cache the merged celebrity slice per user for 30–60 s | Repeat feed opens are free | Staleness up to TTL; memory per active user |
| **6. Shared "celebrity bucket" timelines** | Group celebrities into buckets; precompute merged timeline per bucket; reader merges a few buckets instead of 300 lists | Shared computation across millions of readers | Bucket membership must match what the reader follows → filtering waste |
| **7. Product cap on following** | Hard limit (e.g. 7,500) + follow rate limits | Bounds worst case absolutely | Product constraint, not engineering |

**Recommended combination:** 1 + 2 + 3 by default, 4 for the top heavy-reader cohort, 5 as a cheap latency smoothing layer, 7 as the safety net.

### 12.4 Latency budget (p99 target < 500 ms server-side; illustrative)

```
Gateway + auth                            10 ms
Read timeline:A (ZSET range)               3 ms
Celebrity last-post MGET (300 keys)        3 ms
Pull from ~5–20 active celebrities        10 ms   (parallel, pipelined)
Own-posts read (RYW)                       5 ms   (parallel with above)
k-way merge                               <1 ms
Hydration: post multiget (20 ids)         10 ms   (cache hit ~99%)
           user multiget (≤20 authors)     5 ms
           block/privacy check             5 ms
URL construction (deterministic)          <1 ms
Serialization                              5 ms
────────────────────────────────────────────────
Total p50 ≈ 60 ms; p99 ≈ 150–250 ms with cache-miss and tail effects
```

### 12.5 Tail-latency controls
- **Hedged requests** to a replica after p95 latency elapses on slow shards.
- **Partial results over timeouts:** if a celebrity shard misses its 50 ms budget, return the page without it and mark `next_cursor` so that slice is merged in on the next page. Degrade freshness, never availability.
- **Hot-key replication:** a viral celebrity's `author_recent` list is read by hundreds of millions — replicate the key (read replicas, or a client-side L1 cache with 1–2 s TTL) to avoid a single Redis shard melting.

### 12.6 Fan-out side at scale
- Fan-out tasks are chunked (e.g., 1,000 followers per task) so a 100K-follower author fans out across many workers in parallel.
- Group writes by Redis slot and pipeline → ~1 network round trip per slot per chunk.
- Prioritize fan-out to **currently online** followers first (presence signal), dormant-but-recent second.
- Kafka absorbs bursts (New Year's midnight); fan-out lag is a monitored SLO, with autoscaling on consumer lag.

---

## 13. Deep Dive 5: Uploading Large Media (up to 4 GB)

### 13.1 Constraints that shape the design
- Mobile networks drop; apps get backgrounded or killed mid-upload.
- A single S3 PUT tops out at 5 GB and offers no resume — one failure at 3.9 GB restarts from zero.
- 4 GB over a 20 Mbps uplink ≈ 27 minutes; resumability is mandatory, not nice-to-have.

### 13.2 Options

| Option | Pros | Cons |
|---|---|---|
| Upload through API servers | Inline validation | Bandwidth & connection exhaustion on app tier; no resume |
| Single presigned PUT | Simplest direct upload | No resume; 5 GB hard cap; one TCP stream underutilizes high-latency links |
| **Presigned multipart upload (S3 MPU / GCS resumable)** ✅ | Parallel parts, per-part retry, resume across app restarts, direct to storage | Session state to manage; orphaned parts cost money unless cleaned up |
| Own resumable upload service (tus protocol) | Standard open protocol; full control (inline hashing, quotas) | You now operate a high-bandwidth ingest tier |
| Edge-accelerated ingest (upload to nearest POP, backbone to origin region) | Big win for far-from-origin users (long RTT kills TCP throughput) | Extra cost; vendor-specific |

**Choice:** Presigned multipart, entered through an upload-acceleration edge when the client is far from the storage region.

### 13.3 Multipart upload protocol

```
Client                         Media Service                 Object Store
  │ POST /media/uploads             │                              │
  │ (size, sha256, upload_key) ────▶│ CreateMultipartUpload ──────▶│
  │                                 │◀──────────── UploadId ───────│
  │                                 │ persist UploadSession         │
  │◀── upload_id, part_size=16MB, total_parts=250 ─────────────────│
  │                                                                 │
  │ GET parts?from=1&count=50 ─────▶│ presign 50 part URLs (TTL 1h) │
  │◀──── part URLs ─────────────────│                               │
  │                                                                 │
  │ PUT part 1..50 (4–6 in parallel, each with Content-MD5) ───────▶│
  │◀──────────────────────────────────── ETag per part ─────────────│
  │ (persist completed parts + ETags locally on device)            │
  │                                                                 │
  │ … app killed … relaunch …                                       │
  │ GET /media/uploads/{id} ───────▶│ ListParts ──────────────────▶│
  │◀── completed_parts=[1..37] ─────│◀─────────────────────────────│
  │ resume at part 38                                               │
  │                                                                 │
  │ POST …/complete (part ETags) ──▶│ CompleteMultipartUpload ────▶│
  │                                 │ verify size/checksum          │
  │                                 │ Media → UPLOADED              │
  │                                 │ emit media.uploaded (outbox)  │
  │◀── 202 PROCESSING ──────────────│                               │
```

**Part sizing:** S3 parts must be 5 MB–5 GB with max 10,000 parts. 16 MB parts → 250 parts for 4 GB: small enough that a retry costs little, large enough to keep request overhead low. Adapt dynamically: smaller parts on poor networks, larger on Wi-Fi.

**Parallelism:** 4–6 concurrent parts saturate a mobile uplink and hide per-request RTT; more just contends for bandwidth.

**Integrity:** per-part `Content-MD5`/checksum (storage rejects corrupted parts) + whole-file SHA-256 verified after completion.

**Security:** presigned URLs are scoped to one key, one part, short TTL, and the session enforces declared `size_bytes` + quota; content type is re-validated server-side (magic bytes), never trusted from the header.

### 13.4 Abandoned upload hygiene
- Bucket lifecycle rule: **abort incomplete multipart uploads after N days** (incomplete parts are billed storage and invisible in normal listings — a classic silent cost leak).
- Sweeper job: Media in `UPLOADING` past `expires_at` → `FAILED`; media `READY` but never attached to a post within 7 days → delete.

### 13.5 Video processing pipeline

```
media.uploaded (video)
   │
   ▼
Probe & validate (ffprobe: codec, duration, resolution, rotation) ──▶ reject bad input
   │
   ▼
Safety / policy scan (async, can gate publish)
   │
   ▼
Split at keyframes (GOP boundaries) into ~10 s chunks
   │
   ├──▶ chunk 1 ─┐   Transcode DAG: each chunk × each rendition in parallel
   ├──▶ chunk 2 ─┤   Ladder: 240p/360p/480p/720p/1080p (H.264 baseline for reach)
   ├──▶  …      ─┤   Audio: AAC separately
   └──▶ chunk N ─┘
   │
   ▼
Segment into 2–6 s HLS/DASH segments (CMAF fMP4 → one set serves both)
Write master.m3u8 + per-rendition playlists + poster frame + preview sprite
   │
   ▼
Media → READY (conditional write) → media.ready → Post → PUBLISHED
```

| Decision | Options | Recommendation |
|---|---|---|
| Transcode serially vs chunked-parallel | Serial: simple, ~real-time×N for long videos. Chunked: wall-clock ≈ one chunk's time | **Chunked-parallel** — a 4 GB video goes from tens of minutes to a few minutes |
| Publish when | All renditions done vs fastest rendition done | **Progressive publish:** go live once 360p + 720p exist; higher rungs arrive later and are added to the master playlist |
| Codec | H.264 everywhere vs HEVC/AV1 | **H.264 for everyone first**; re-encode to AV1/HEVC only when view count crosses a threshold (encode cost ↔ egress savings) |
| Compute | CPU vs GPU/ASIC encoders | Mix: GPU for throughput, CPU for quality-sensitive popular content |

Chunk workers are idempotent (output key = `media_id/rendition/chunk_n`), so retries after a worker crash simply overwrite. See *YouTube Video Streaming* guide for ladder design and *File Upload & Malware Scanning* for the scanning tier.

---

## 14. Deep Dive 6: Fast Global Media Delivery

### 14.1 Layered caching

```
 Client cache (disk/memory, immutable URLs → cache forever)
      │ miss
      ▼
 CDN edge POP (hundreds of cities; anycast/GeoDNS routes to nearest healthy POP)
      │ miss
      ▼
 Regional shield / mid-tier cache (collapses misses from many edges into one)
      │ miss
      ▼
 Origin: object store in nearest replica region
```

| Option | Pros | Cons |
|---|---|---|
| Serve from origin regions only | Simple | Cross-ocean RTTs (150–300 ms) per request; egress cost; origin melts on virality |
| **Single third-party CDN** | Fast to adopt; global footprint | Vendor lock-in and outage blast radius |
| **Multi-CDN + own edge/ISP-embedded caches** ✅ (at mega scale) | Resilience, cost leverage, deep ISP placement | Complex steering, inconsistent cache behavior across vendors |

Start with one CDN + origin shield; evolve to multi-CDN with real-user-measurement-based traffic steering.

### 14.2 Make every media URL immutable

```
https://cdn.example.com/m/{media_id}/{rendition}/{content_hash}.avif
Cache-Control: public, max-age=31536000, immutable
```

- Content-addressed URLs mean **no cache invalidation** is ever needed for updates — a new version is a new URL.
- Deletion is the exception: remove from origin + issue a CDN purge; the feed stops referencing it immediately (metadata path), so residual edge copies are unreachable except by direct URL until purged.

### 14.3 Images: render fast on any device

| Technique | Effect |
|---|---|
| Multiple widths (150/320/640/1080) + client picks by viewport & DPR | Don't ship 1080w to a thumbnail slot |
| Format negotiation: AVIF → WebP → JPEG | 30–50% smaller than JPEG at similar quality |
| **Blurhash / LQIP inlined in feed JSON** | Instant colored placeholder, zero extra requests — perceived latency ~0 |
| Prefetch next 3–5 feed items' images on scroll | Item is cached before it enters the viewport |
| HTTP/2 or HTTP/3 (QUIC) connection reuse | Avoid per-image handshakes; QUIC handles lossy mobile networks better |

### 14.4 Video: first frame fast, no rebuffering

| Technique | Effect |
|---|---|
| Adaptive bitrate (HLS/DASH), 2–6 s segments | Player matches network; switches on throughput changes |
| **Start at a low rung** then ramp up | First frame in < 1 s even on 3G |
| **Prefetch first segment + poster of the next video** in feed | Scroll-to-play feels instant |
| Short first segment (1–2 s) | Less to download before playback starts |
| Popular videos pre-positioned to edges ("push" warm-up) | Viral content never cold-misses globally |
| Byte-range / segment caching at edges | Long videos cached in pieces; users rarely watch to the end |

### 14.5 Origin & multi-region
- Uploads land in the region nearest the uploader; **async cross-region replication** for durability and locality.
- Hot set (recent, popular) replicated to all regions proactively; long tail replicated lazily or fetched cross-region through the shield on first miss.
- **Request coalescing** at shield and edge: 10,000 simultaneous misses for a viral post become **one** origin fetch (the media equivalent of cache-stampede protection).
- Storage tiering: hot → standard; > 90 days with low access → infrequent-access tier; originals of long-tail content → archive (keep renditions hot).

### 14.6 Private accounts and media authorization

| Option | Pros | Cons |
|---|---|---|
| Public unguessable URLs | Max cache hit rate; zero edge logic | Leaked URL viewable indefinitely |
| **Signed URLs/cookies with short TTL (e.g., hours), signature excluded from cache key** ✅ | Edge validates token; cache shared across viewers | Revocation granularity = TTL; signing on every feed response |
| Per-request auth to origin | Instant revocation | Destroys CDN effectiveness |

The Feed Service decides *whether* a viewer may see a post (CP check); the signed URL just carries that decision to the edge for a bounded time.

### 14.7 What to measure
Per-region p50/p95 image time-to-render, video time-to-first-frame, rebuffer ratio, CDN hit ratio (target > 95% bytes), shield hit ratio, origin egress, per-POP error rate. Steer traffic away from POPs/CDNs that degrade.

---

## 15. CAP / PACELC Positioning

| Component | Choice | Reasoning |
|---|---|---|
| Timeline cache (Redis) | **AP / EL** | Stale or partially missing feed is fine; rebuildable from source |
| Post metadata (Cassandra) | **AP / EL** (`LOCAL_QUORUM` writes) | Availability of reads > instant global visibility |
| Follow edges & counters | **AP / EL** | A follow appearing a few seconds late is harmless; counters reconciled |
| Media object store | **AP** (read-after-write for new keys) | Immutable, content-addressed objects never conflict |
| CDN | **AP** | Serve cached bytes during origin partition |
| **Block list & privacy checks** | **CP / EC** | A leaked post to a blocked user is a trust/safety incident — prefer an error to an incorrect allow |
| **Post / media state machine transitions** | **CP** (conditional writes / LWT) | Prevent double publish, publish-before-ready |
| **Idempotency keys** | **CP** | Duplicate posts on retry are user-visible bugs |
| Upload sessions | **CP** for completion | Only one `CompleteMultipartUpload` should win |

Most of the system is AP/EL; the **named CP exceptions** are privacy enforcement and lifecycle/idempotency mutations. (Note: the "quorum" in `LOCAL_QUORUM` is replication overlap, not a consensus vote — see *CAP/PACELC* guide.)

---

## 16. Failure Modes & Mitigations

| Failure | Symptom | Detection | Mitigation |
|---|---|---|---|
| Celebrity posts (100M+ followers) | Fan-out queue explodes; lag for everyone | Fan-out lag metric | Hybrid: celebrities are pull-only; per-author fan-out budget |
| Hot celebrity read key | One Redis shard CPU-bound | Per-key QPS / shard CPU | Replicate hot keys; client L1 cache (1–2 s TTL) |
| Fan-out worker crash mid-batch | Some followers miss a post | Kafka consumer lag; offsets uncommitted | At-least-once + idempotent ZADD; commit offset after batch |
| DB write succeeds, event lost | Post published but never in feeds | Reconciliation: PUBLISHED posts with no fan-out record | Transactional outbox / CDC instead of dual write |
| Redis timeline node loss | Feeds empty for a shard of users | Cache miss rate spike | Replicas + failover; cold rebuild-by-pull with rate limit to protect Post DB |
| Rebuild stampede after cache loss | Post DB overloaded by millions of rebuilds | DB latency, miss rate | Request coalescing per user; prioritize active users; serve degraded "recent posts" feed |
| Upload interrupted at 90% | User restarts 4 GB upload | Client retries, session state | Multipart resume via ListParts; local part journal |
| Orphaned multipart parts | Rising storage bill with no visible objects | Storage inventory reports | Lifecycle abort rule; sweeper |
| Transcoding backlog (viral event) | Posts stuck in PROCESSING | Queue depth, p95 processing time | Autoscale; priority queues (short videos, verified creators first); progressive publish |
| Corrupt / malicious upload | Decoder crash, decompression bomb | Worker crash loop on same media | Sandbox decoders; size/dimension limits; poison-message DLQ after N attempts |
| Deleted post still in timelines | Ghost items | Hydration returns tombstones | Filter at hydration; lazy removal |
| Block not yet propagated | Blocked user sees content | — (must prevent, not detect) | Read-time CP block check; never rely on timeline state |
| CDN POP / vendor outage | Regional media failures | RUM error rate per POP | Multi-CDN steering; DNS failover; stale-while-revalidate |
| Origin region outage | Cache-miss media unavailable | Origin 5xx | Cross-region replicas; shield fails over to secondary origin |
| Clock skew on ID generators | Out-of-order post IDs → feed order glitches | Monotonicity check per generator | Snowflake refuses to issue IDs if clock moves backward; NTP |
| Snowflake ID used as ZSET score | Feed order/cursor corruption | Duplicate/missing items on paginate | Score = ms timestamp, member = post_id |

**Pattern class:** celebrity fan-out and hot media keys are this domain's version of the crawler's spider trap — a single entity whose degree is orders of magnitude above the mean, requiring a *separate code path*, not just more capacity.

---

## 17. Scalability & Sharding

| Data | Shard key | Why | What becomes expensive |
|---|---|---|---|
| Posts | `author_id` | Profile grid is one partition scan | Lookup by `post_id` → separate `post_by_id` table + cache |
| `following` | `follower_id` | "Who do I follow" for feed merge & rebuild | Reverse lookup → separate table |
| `followers` | `followee_id` | Fan-out reads one partition (paged) | Celebrity partitions are huge → page with clustering + split into sub-partitions (`followee_id, bucket`) |
| Timelines | `user_id` (Redis Cluster hash slot) | Feed read is one key | Cross-slot ops impossible → keep per-user ops single-key |
| Media metadata | `media_id` | Point lookups | — |
| Object store | key prefix hashing (`media_id` first) | Avoid hot prefixes | Listing by user → use metadata DB, never bucket listing |
| Kafka `post.published` | `author_id` | Ordering per author | Celebrity partition skew → they don't fan out via this path anyway |
| Kafka fan-out tasks | `follower_shard` | Batch writes per Redis node | — |

**Hot partition note:** a 600M-follower account makes `followers` partition for that user enormous. Bucket it: `PRIMARY KEY ((followee_id, bucket), follower_id)` with `bucket = hash(follower_id) % 1024` for huge accounts — the same "split hot keys into sub-queues" move as the crawler's hot domain fix.

**Horizontal scaling plan**

| Tier | Scaling lever |
|---|---|
| Stateless services (API, Feed, Post, Graph, Media) | Autoscale on CPU/QPS |
| Fan-out & processing workers | Autoscale on Kafka consumer lag |
| Redis Cluster | Add shards; resharding moves hash slots |
| Cassandra | Add nodes to ring; RF=3 per region |
| Object store / CDN | Managed; capacity planning via egress forecast |

Multi-region: active-active for reads in every region; writes for a user routed to their **home region** to avoid conflicting cross-region edits of the same entity; async replication between regions.

---

## 18. Senior vs Staff Answer Differentiators

### Senior-level expectations
- Identifies post, media, graph, feed, CDN components
- Uses presigned URLs to S3 for uploads
- Describes push vs pull and picks hybrid
- Puts a CDN in front of media
- Basic capacity math

### Staff-level expectations (additional)

| Topic | What Staff adds |
|---|---|
| Media architecture | Explicit **control-plane vs data-plane** split as the anchor constraint |
| Post creation | State machine with conditional transitions; transactional outbox; idempotency keys |
| Following | Dual-direction storage with single source of truth + CDC; timelines are a cache, never authorization |
| Feed | Distinguishes high-follower (write amp) vs high-following (read amp) problems; per-edge push override |
| Correctness traps | ZSET double precision vs 64-bit IDs; offset pagination drift; dual-write event loss |
| Large uploads | Multipart part sizing math; resume via ListParts; lifecycle abort of orphaned parts; progressive publish |
| Global delivery | Immutable content-addressed URLs (no invalidation); shield + request coalescing; LQIP; first-segment prefetch; signed URLs with cache-key hygiene |
| Cost | Client-side resize, deferred AV1 re-encode for popular content, storage tiering |
| CAP | Per-component AP/EL with named CP exceptions (blocks/privacy, state transitions, idempotency) |
| Operations | Rebuild stampede, celebrity hot keys, transcoding backlog — raised **before** being asked |

**The single most important Staff move:** separating the **two degree problems** (celebrities vs heavy followers) and showing that the hybrid solves one while a distinct set of techniques solves the other.

---

## 19. Interview Time Allocation

For a 45-minute session:

| Phase | Time | Focus |
|---|---|---|
| Requirements & scope | 4 min | FR1–FR5, park ranking/stories; privacy + RYW as NFRs |
| Capacity estimation | 4 min | 58K feed reads/s, 231K internal fan-out/s, 1.4 Tbps egress |
| Entities & APIs | 4 min | Snowflake IDs, cursor pagination, 202 + idempotency |
| High-level design | 6 min | Draw both planes; walk FR1–FR3 through the diagram |
| Feed deep dive (Q3 + Q4) | 12 min | Push/pull/hybrid, read path, heavy-follower techniques, latency budget |
| Large upload (Q5) | 6 min | Multipart protocol, resume, transcoding DAG |
| Global delivery (Q6) | 5 min | CDN tiers, immutable URLs, ABR, LQIP/prefetch |
| Failure modes & CAP | 4 min | Celebrity, rebuild stampede, outbox, block CP |

**Cut if short:** follow-graph storage alternatives, codec strategy. **Never cut:** hybrid feed + heavy-follower answer, control/data plane split.

---

## 20. Quick-Reference Cheatsheet

```
KEY NUMBERS (illustrative)
──────────────────────────
500M DAU · 100M posts/day      → ~1.2K posts/s (peak ~3.5K)
5B feed opens/day              → ~58K feed reads/s (peak ~150K)
20B timeline inserts/day       → ~231K internal writes/s
~680 TB/day originals          → ~63 Gbps ingest
~15 PB/day egress              → ~1.4 Tbps avg → CDN mandatory
Timelines: 500 entries × 16 B × 500M users ≈ 4 TB (fits in Redis)

FEED
────
Hybrid: push for normal authors, pull for celebrities (> ~100K followers)
Skip fan-out to users inactive > 30 days; rebuild-by-pull on return
Heavy followers: last-post-ts filter → pipelined batch pull → k-way heap merge
                 + per-edge push override for heavy readers
Redis ZSET: score = ms timestamp, member = post_id (NOT post_id as score)
Cursor pagination on (ts, post_id); RYW by merging own posts

UPLOAD
──────
Presigned multipart, 16 MB parts (5 MB–5 GB, ≤10,000 parts), 4–6 parallel
Resume = ListParts + local part journal; per-part checksum
Lifecycle rule aborts incomplete MPUs; sweeper for orphans
Video: split at GOPs → parallel transcode → CMAF HLS/DASH ladder
Progressive publish; AV1 only for popular videos

DELIVERY
────────
Client cache → CDN edge → regional shield → multi-region origin
Immutable content-addressed URLs, max-age=1y, no invalidation
AVIF/WebP + width variants + blurhash in feed JSON
ABR, low starting rung, prefetch first segment of next video
Signed URLs (short TTL) for private content; token not in cache key

CAP
───
AP/EL: timelines, posts, follows, counters, media, CDN
CP:    block/privacy checks, post/media state transitions, idempotency, upload completion

STAFF SIGNALS
─────────────
✓ Control plane vs data plane as anchor constraint
✓ Two degree problems: high-follower vs high-following
✓ Timelines are cache, never authorization
✓ Outbox/CDC instead of dual writes
✓ ZSET precision trap
✓ Orphaned multipart cost leak
✓ Request coalescing at shield for viral media
✓ Per-component CAP with named CP exceptions
```

---

## 21. Related Guides

| Guide | Relationship |
|---|---|
| *News Feed System* | General fan-out mechanics; ranking extensions beyond chronological |
| *Twitter / Fan-out Architecture* | Celebrity problem in depth; same hybrid push/pull pattern |
| *YouTube Video Streaming* | Transcoding ladder, ABR, chunked parallel encode |
| *File Upload & Malware Scanning* | Upload validation & scanning tier hooked into Media Processing |
| *Dropbox* | Control-plane vs data-plane split; chunked resumable upload |
| *Distributed Unique ID Generation* | Snowflake `post_id` design and clock-skew handling |
| *Sharding Strategies* | Shard-key selection and hot-partition bucketing |
| *Apache Kafka* | Event bus durability (`acks` + `min.insync.replicas`), consumer lag back-pressure |
| *CAP / PACELC Theorem* | Per-component positioning; replication vs consensus quorum |
| *Distributed Key-Value Store (Dynamo-style)* | Cassandra-style tunable consistency used for posts and graph |
| *Notification System* | Follow-request and "new follower" notifications |
| *Web Crawler* | Spider traps ↔ celebrity fan-out as the same "extreme-degree entity" pattern class |

---

*Guide compiled for Senior & Staff-level system design interview preparation.*
*Topics: Instagram, Media Upload, Hybrid Fan-out, Social Graph, CDN, Adaptive Bitrate Streaming.*
