# Tinder — System Design Interview Guide
### Senior & Staff Engineer Level

> All numbers in this guide are **illustrative** interview-grade estimates, not Tinder's published figures. Decimal (SI) units throughout.

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
9. [HLD Walkthrough: 4 Functional Requirements in Detail](#9-hld-walkthrough-4-functional-requirements-in-detail)
10. [Deep Dive 1: Consistency — Exactly One Match, Never a Missed Match](#10-deep-dive-1-consistency--exactly-one-match-never-a-missed-match)
11. [Deep Dive 2: Low Latency — Sub-300 ms Recommendation Stacks](#11-deep-dive-2-low-latency--sub-300-ms-recommendation-stacks)
12. [Deep Dive 3: Scale & Durability — 175K Swipes/s Peak](#12-deep-dive-3-scale--durability--175k-swipess-peak)
13. [Deep Dive 4: Never Show a Profile Twice](#13-deep-dive-4-never-show-a-profile-twice)
14. [Cross-Cutting: Location Privacy & Safety](#14-cross-cutting-location-privacy--safety)
15. [CAP / PACELC Positioning](#15-cap--pacelc-positioning)
16. [Failure Modes & Mitigations](#16-failure-modes--mitigations)
17. [Scalability & Sharding](#17-scalability--sharding)
18. [Senior vs Staff Answer Differentiators](#18-senior-vs-staff-answer-differentiators)
19. [Interview Time Allocation](#19-interview-time-allocation)
20. [Quick-Reference Cheatsheet](#20-quick-reference-cheatsheet)
21. [Related Guides](#21-related-guides)

---

## 1. Problem Statement & Scope

Tinder is a location-based dating app. Users build a profile, set preferences (age range, gender, max distance), are shown a "stack" of nearby candidate profiles one at a time, and swipe **right** (like) or **left** (pass). When two users both swipe right on each other, a **match** is created, both are notified, and they can chat.

### In scope
- Profile + preferences (incl. photos and location)
- Recommendation stack of nearby, preference-compatible users
- Swiping (like / pass)
- Match detection and notification

### Out of scope (call out, then park)
- Chat after match → *Messaging Application* guide
- Super Likes, Boosts, Passport (travel mode), paid tiers — mention only as levers
- ML ranking model internals, trust & safety review tooling, payments

### What the interviewer is really testing

| Signal | What it looks like |
|---|---|
| Two-party correctness | The **mutual-swipe race** raised unprompted, and a partitioning scheme that makes the match check atomic |
| Geo + multi-predicate search | Geospatial index *plus* reciprocal preference filtering, with an honest view of query fan-out |
| Write-heavy hot path | Tens of thousands of swipes/s, each tiny; durability of every "like" |
| Dedup at scale | Never re-showing swiped profiles without an unbounded exclusion list |
| Privacy | Location fuzzing against trilateration; blocks enforced on read |

**Anchor constraint (drives everything downstream):** *A swipe is a single-user write, but a match is a two-user transaction.* If swipes are partitioned by swiper, the match check crosses partitions and races. The Staff move is to choose a data layout where the two halves of a potential match **land on the same key**, turning a distributed transaction into a single-item atomic update.

---

## 2. Functional Requirements

| # | Requirement | Notes |
|---|---|---|
| FR1 | Users create a profile with photos and set preferences | Age range, interested-in genders, max distance; location updated from device |
| FR2 | Users get a stack of potential matches near them that fit their preferences | Reciprocal: candidate must also fit *their* preferences of you |
| FR3 | Users swipe right (like) or left (pass) on a profile | One decision per pair; works through flaky networks |
| FR4 | Users are notified when a mutual like creates a match | Second swiper sees "It's a Match!" instantly; first swiper gets a push |
| FR5 | Users view their list of matches and can unmatch / block | Feeds the chat system (out of scope) |

FR1–FR4 are walked through in detail in §9.

---

## 3. Non-Functional Requirements

| Requirement | Target | Why it matters |
|---|---|---|
| **Consistency of matching** | Exactly one match per mutual like; **zero missed matches**; idempotent under retries | A missed match is invisible product failure; a duplicate match is a visible bug |
| **Low latency** | Stack fetch p99 < 300 ms; swipe ack p99 < 100 ms; match shown instantly to second swiper | Swiping is a rapid loop; any stall breaks the experience |
| **Scale & durability** | 50M DAU, ~58K swipes/s avg, ~175K/s peak; an acknowledged swipe is never lost | Lost like = lost potential match |
| **No repeats** | A swiped profile never reappears (within a retention policy); served-but-unswiped not reshown for days | Repeats feel broken and waste the stack |
| Availability | 99.99% for stack + swipe; degrade "instant match" before degrading swipe acceptance | Users swipe during peaks (evenings, Sundays) |
| Freshness | New profiles discoverable within minutes; location changes reflected within minutes | Stale location = wrong-city candidates |
| Privacy & safety | Exact location never exposed (directly or via trilateration); blocks effective immediately | Physical-safety stakes, not just data hygiene |

**Read:write profile:** swipes make this unusually **write-heavy** at the metadata layer (~58K writes/s vs ~3.5K stack reads/s), while photo delivery is read-heavy and CDN-dominated.

---

## 4. Capacity Estimation

```
Assumptions (illustrative):
  Registered users                  = 300M
  DAU                               = 50M
  Discoverable profiles (active 30d)= 100M
  Swipes / DAU / day                = 100      (40% right, 60% left)
  Reciprocal-like rate              = 2% of likes
  Stack fetches / DAU / day         = 6        (50 profiles per page)
  Photos / profile                  = 6

Swipes (client-facing writes):
  50M × 100                         = 5B swipes/day
  5B / 86,400                       ≈ 58K swipes/s  (peak ~3× ≈ 175K/s)
  Likes: 40% × 5B                   = 2B/day ≈ 23K/s (each needs a match check)

Matches:
  2B × 2%                           = 40M matches/day ≈ 460/s (peak ~1.4K/s)
  Push notifications                ≈ 40M/day (to the first swiper only)

Stack reads:
  50M × 6                           = 300M/day ≈ 3.5K/s (peak ~10K/s)
Candidate generation (stack refills, internal):
  ~1 refill per 200 swipes + invalidations ≈ 100M/day ≈ 1.2K/s
  each = 1 geo query returning ~1,000 candidates → filter → rank → keep 200

Location updates:
  50M × 4 meaningful updates/day    = 200M/day ≈ 2.3K/s
  reindex only on grid-cell change (~30%) ≈ 700 index writes/s

Swipe storage (two tables — pair table + per-swiper history):
  ~100 B/row stored × 5B × 2        ≈ 1 TB/day   × RF3 ≈ 3 TB/day ≈ 1.1 PB/yr
  → left swipes (60%) get a TTL (e.g., 90 days) — biggest storage lever

Dedup Bloom filters (per active user):
  10K retained swipes × 9.6 bits    ≈ 12 KB/user
  × 50M DAU                         ≈ 600 GB  (Redis cluster, sharded by user)

Recommendation index:
  100M docs × ~1 KB                 ≈ 100 GB primary
  → small! The problem is query QPS × shard fan-out, not data size

Photos:
  Originals: 100M × 6 × 2 MB        ≈ 1.2 PB
  Renditions: 100M × 6 × 4 × 150 KB ≈ 360 TB
  Egress: 5B swipes × ~1.5 photos × 150 KB ≈ 1.1 PB/day ≈ 104 Gbps avg
          (peak ~300 Gbps) → CDN mandatory
```

**Key insights**
1. **Swipes are the write hot path** — 175K/s peak, but each is ~100 bytes. This is a throughput-and-atomicity problem, not a storage-bandwidth problem.
2. **The recommendation index is tiny (100 GB)** — it fits on one machine. You shard it for *query throughput and locality*, not capacity. Saying this out loud is a Staff signal.
3. **Left swipes dominate storage** and have low long-term value → lifecycle policy, not more disks.
4. **Photo egress dominates bandwidth** by orders of magnitude → object store + CDN (reuse the *Instagram* guide's media plane).

---

## 5. Core Entities

```
User
  user_id (PK, 64-bit Snowflake), phone/email (auth), status
    (ACTIVE | HIDDEN | BANNED | DELETED), created_at, last_active_at

Profile
  user_id (PK), name, birthdate (→ age), gender, bio,
  photo_ids[] (ordered), verified, updated_at

Preferences
  user_id (PK), age_min, age_max, interested_in[] (genders),
  max_distance_km, prefs_version

Location (private, encrypted at rest)
  user_id (PK), precise_lat, precise_lng, accuracy, updated_at,
  grid_cell (e.g., H3/S2 cell — the ONLY location used for search),
  geoshard_id

Swipe (canonical pair record — source of truth for matching)
  pair_id = "{lo_id}:{hi_id}"   (lo_id = min(a,b), hi_id = max(a,b))
  lo_decision, lo_ts, lo_swipe_id,  hi_decision, hi_ts, hi_swipe_id

SwipeHistory (derived, per swiper — for dedup & "who did I swipe")
  (swiper_id, swipee_id, decision, ts)   partition: swiper_id  TTL on PASS

Match
  match_id = hash(lo_id, hi_id)  (deterministic → idempotent creation)
  lo_id, hi_id, created_at, status (ACTIVE | UNMATCHED | BLOCKED)
UserMatches (derived index)
  (user_id, created_at DESC, match_id, other_user_id)

RecommendationStack (cache, derived — never a source of truth)
  key: stack:{user_id} → [candidate_id …] (~200), generated_at,
       gen_cell, gen_prefs_version

Block
  (blocker_id, blocked_id, created_at) — stored in both directions
```

**Why a deterministic `match_id`?** `hash(lo_id, hi_id)` means any retry, replay, or reconciliation job that tries to create the match produces the *same* key → a put-if-absent makes match creation idempotent for free. See *Distributed Unique ID Generation* for why random IDs would need a separate dedupe table here.

---

## 6. API / System Interface

All endpoints authenticated; `user_id` comes from the token, never the body.

### Profile, preferences, location

```http
PUT  /v1/profile
{ "name":"Ana", "birthdate":"1996-04-02", "gender":"F", "bio":"…" }
→ 200 { "user_id":"u_1", "status":"ACTIVE" }

POST /v1/profile/photos               # control plane only; bytes go direct to object store
{ "content_type":"image/jpeg", "size_bytes":1800000, "client_upload_key":"uuid" }
→ 201 { "photo_id":"ph_9", "upload_url":"https://…presigned…" }
PUT  /v1/profile/photos/order  { "photo_ids":["ph_9","ph_3", …] }

PUT  /v1/preferences
{ "age_min":25, "age_max":35, "interested_in":["M"], "max_distance_km":25 }
→ 200 { "prefs_version": 7 }

POST /v1/location
{ "lat":47.61, "lng":-122.20, "accuracy_m":30 }
→ 204                                 # server snaps to grid cell; never echoes coordinates
```

### Recommendations

```http
GET /v1/recommendations?limit=50
→ 200 {
  "rec_token": "r_abc",               # ties swipes back to the serving batch (analytics, dedup)
  "profiles": [{
     "user_id":"u_77", "name":"Sam", "age":29,
     "distance": "3 km away",         # rounded, never exact
     "bio":"…",
     "photos":[{ "placeholder":"<blurhash>",
                 "urls":{"640w":"https://cdn…","1080w":"https://cdn…"} }]
  }, …]
}
```

### Swipes

```http
POST /v1/swipes
{ "target_id":"u_77", "decision":"LIKE", "client_swipe_id":"uuid", "rec_token":"r_abc" }
→ 200 { "matched": true, "match_id":"m_5f2…" }      # or { "matched": false }

POST /v1/swipes:batch                 # offline / backgrounded queue flush
{ "swipes":[{…}, {…}] }
→ 200 { "results":[{ "client_swipe_id":"…", "matched":false }, …] }
```

### Matches

```http
GET    /v1/matches?cursor=<ts,match_id>&limit=50
DELETE /v1/matches/{match_id}          # unmatch
POST   /v1/blocks  { "target_id":"u_77" }
```

**Design choices to defend**
- **Swipe returns `matched` synchronously** — this is what puts the match check on the request path and makes Deep Dive 1 necessary. (An async-only design is simpler but loses the instant "It's a Match!" moment — see §10 options.)
- **`client_swipe_id` idempotency** — the client queues swipes locally and retries; a retried LIKE must not change state or double-notify.
- **Recommendations return hydrated profiles + CDN URLs + blurhash** — the client prefetches photos for the next several cards so swiping never waits on the network.
- **Location is write-only from the client's perspective** — no endpoint ever returns coordinates, only a rounded distance string (§14).

---

## 7. Data Flow

### 7.1 Create profile (FR1)
```
1. PUT /profile, PUT /preferences          → Profile DB (Postgres)
2. POST /profile/photos                    → presigned URL; client uploads to object store
3. media.uploaded → Photo pipeline         → validate, strip EXIF, safety scan, face check,
                                             renditions + blurhash → photo READY
4. POST /location                          → Location Service snaps to grid cell, picks geoshard
5. CDC (profile/prefs/location changes)    → Indexer → upsert doc into geoshard of the
                                             recommendation index (searchable in seconds–minutes)
```

### 7.2 Get recommendations (FR2)
```
1. GET /recommendations
2. Rec Service reads stack:{A} (Redis); if stale/empty → synchronous generation (slow path)
3. Serve-time filters: swiped (Bloom + recent-swipe set), served-recently, blocked,
                       banned/hidden/deleted, prefs-version mismatch
4. Hydrate 50 profiles (profile cache) + build CDN photo URLs + rounded distances
5. If stack below low-water mark → emit stack.refill (async)
```

### 7.3 Swipe (FR3)
```
1. POST /swipes (client_swipe_id)
2. Swipe Service: atomic update of pair record (only A's side) → returns post-image
3. If A=LIKE and other side=LIKE → Match Service: create match (put-if-absent)
4. Respond { matched } ; CDC/outbox → swipe.recorded → SwipeHistory, Bloom update,
   analytics/ML feature stream
```

### 7.4 Match notification (FR4)
```
1. match.created (outbox) → Kafka
2. Match index worker → UserMatches rows for both users
3. Notification Service → push (APNs/FCM) to the user who swiped FIRST;
                          in-app/WebSocket event to both if online
4. Chat Service → creates conversation (out of scope)
```

---

## 8. High-Level Design

```
                             ┌───────────────────────────────────────┐
                             │           Mobile Client               │
                             │ local swipe queue · photo prefetch    │
                             └───────┬───────────────────────┬───────┘
                     metadata (HTTPS)│                       │ photos (download / presigned upload)
                                     ▼                       ▼
                       ┌──────────────────────┐   ┌──────────────────────────────┐
                       │  API Gateway / LB    │   │ CDN (edge → shield) ◀──────┐  │
                       │ auth · rate limits   │   └──────────────────────────────┘
                       └──┬──────┬──────┬───┬─┘                                │
         ┌────────────────┘      │      │   └───────────────────┐              │
         ▼                       ▼      ▼                       ▼              │
┌────────────────┐  ┌──────────────┐ ┌──────────────────┐ ┌──────────────┐   │
│ Profile Service│  │ Location Svc │ │ Recommendation   │ │ Swipe Service│   │
│ profile, prefs,│  │ snap to grid,│ │ Service          │ │ atomic pair  │   │
│ photo sessions │  │ geoshard pick│ │ serve · generate │ │ update       │   │
└──┬─────────┬───┘  └──────┬───────┘ └──┬────┬────┬─────┘ └──┬────────┬──┘   │
   │         │             │            │    │    │          │        │      │
   ▼         ▼             ▼            │    │    │          ▼        ▼      │
┌────────┐ ┌───────────┐ ┌──────────┐  │    │    │   ┌────────────┐ ┌──────────────┐
│Profile │ │ Object    │ │ Location │  │    │    │   │ Swipe Pair │ │ Match Service│
│DB (PG, │ │ Store     │─┘ Store    │  │    │    │   │ Store      │ │ put-if-absent│
│sharded)│ │ photos    │ │(encrypted)│ │    │    │   │ (DynamoDB/ │ │ Match DB     │
└───┬────┘ └─────┬─────┘ └────┬─────┘  │    │    │   │ Cassandra) │ └──────┬───────┘
    │ CDC        │ media.uploaded│ CDC  │    │    │   └─────┬──────┘        │ outbox
    ▼            ▼               ▼      │    │    │         │ CDC           ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                               Kafka (event bus)                                   │
│ profile.changed · location.changed · swipe.recorded · match.created · stack.refill│
└───┬──────────────┬───────────────┬──────────────────┬───────────────┬────────────┘
    ▼              ▼               ▼                  ▼               ▼
┌─────────┐ ┌─────────────┐ ┌───────────────┐ ┌───────────────┐ ┌──────────────────┐
│ Indexer │ │ Photo       │ │ Swipe-history │ │ Stack Builder │ │ Notification Svc │
│ → geo-  │ │ pipeline    │ │ + Bloom       │ │ workers       │ │ APNs/FCM/WebSocket│
│ shards  │ │ (resize,    │ │ updater       │ │ (generate +   │ │ + UserMatches    │
└────┬────┘ │ scan, EXIF) │ └──────┬────────┘ │ rank)         │ │ index            │
     ▼      └─────────────┘        ▼          └──────┬────────┘ └──────────────────┘
┌────────────────────────┐ ┌─────────────────────┐   │
│ Recommendation Index   │ │ SwipeHistory (Cass.,│   │
│ Elasticsearch, GEO-    │◀┤ by swiper) + Redis  │   │
│ SHARDED by cell groups │ │ Bloom filters       │   │
└───────────┬────────────┘ └─────────────────────┘   │
            └─────────── candidate query ◀───────────┘
                                                      ▼
                                    ┌──────────────────────────────────┐
                                    │ Stack Cache (Redis): stack:{uid}  │
                                    │ served:{uid} ZSET · recent_swipes │
                                    └──────────────────────────────────┘
```

### Component responsibilities & why

| Component | Responsibility | Why this choice |
|---|---|---|
| API Gateway | AuthN, per-user swipe rate limits, routing | Bot swiping is the main abuse vector; stop it at the edge |
| Profile Service + Postgres | Profile, preferences, photo metadata | Low write rate, relational validation, transactions; shard by `user_id` |
| Location Service | Accepts device location, snaps to grid, assigns geoshard | Single choke point where precise location is handled → privacy boundary |
| Recommendation Index (ES) | Geo + multi-predicate candidate retrieval | Geo filters + range filters + scoring in one engine; geosharded for locality |
| Stack Builder | Generate → filter → rank → write top ~200 to cache | Moves expensive work off the request path |
| Stack Cache (Redis) | Per-user precomputed candidate list | Serving becomes a list pop + hydration |
| Swipe Service + Pair Store | Atomic per-pair decision record; detects mutual like | Pair key co-locates both halves of a match (anchor constraint) |
| Match Service | Idempotent match creation; outbox event | Deterministic `match_id` + put-if-absent |
| SwipeHistory + Bloom | Per-swiper history for dedup | Derived from pair store via CDC; read-optimized for "have I seen X" |
| Kafka | Decouples swipe/match from indexing, notifications, ML | Replay, back-pressure, absorbs Sunday-night spikes (see *Kafka* guide) |
| Notification Service | Push + in-app delivery of matches | Reuses *Notification System*; at-least-once with dedupe on `match_id` |
| CDN + Object Store | Photo delivery | ~100 Gbps egress (see *Instagram* guide media plane) |

### Functional requirement coverage

| FR | Path through the diagram |
|---|---|
| FR1 profile | Client → Profile Svc → PG; photos → Object Store → Photo pipeline; Location Svc → CDC → Indexer → ES geoshard |
| FR2 stack | Client → Rec Svc → Stack Cache (→ Stack Builder → ES on refill) → hydration → CDN URLs |
| FR3 swipe | Client → Swipe Svc → Pair Store (atomic) → CDC → SwipeHistory/Bloom |
| FR4 match | Swipe Svc → Match Svc → outbox → Kafka → Notification Svc + UserMatches |
| FR5 matches list | Client → Match Svc → UserMatches (partition `user_id`) |

---

## 9. HLD Walkthrough: 4 Functional Requirements in Detail

### 9.1 FR1 — Create profile with photos and preferences

```
Client ──PUT /profile──▶ Profile Svc ──▶ Profile DB (PG)          status=INCOMPLETE
Client ──POST /photos──▶ Profile Svc ──▶ presigned URL
Client ──PUT bytes─────────────────────▶ Object Store ──media.uploaded──▶ Photo pipeline
                                                         validate · EXIF strip · safety scan
                                                         · face present? · renditions · blurhash
                                         Photo READY ◀──────────────────────────┘
Client ──PUT /preferences─▶ Profile Svc ─▶ PG (prefs_version++)
Client ──POST /location──▶ Location Svc ─▶ Location Store (precise, encrypted)
                                         └▶ grid_cell + geoshard_id
When ≥1 photo READY + prefs + location → status=ACTIVE
CDC ─▶ Indexer ─▶ upsert doc into geoshard(grid_cell)
```

**Index document (what the search engine sees):**
```json
{ "user_id":"u_77", "cell_center": {"lat":47.612,"lon":-122.204},
  "age":29, "gender":"M",
  "pref_age_min":24, "pref_age_max":34, "pref_genders":["F"], "pref_max_km":30,
  "last_active_at":"…", "status":"ACTIVE", "desirability_bucket":4, "verified":true }
```
Notice the doc stores the candidate's **own preferences** too — that's what makes reciprocal filtering (§9.2) a single query.

| Decision | Options | Choice |
|---|---|---|
| Profile store | Postgres sharded by `user_id` / DynamoDB / Mongo | **Postgres** — low write rate (profile edits are rare), strong validation, easy CDC (logical replication / Debezium) |
| Index sync | Dual write from service / CDC | **CDC** — dual write loses updates on partial failure (same reasoning as Instagram's outbox) |
| Photo upload | Through API / presigned direct | **Presigned direct**; photos are small, single PUT suffices (no multipart needed, unlike Instagram's 4 GB video) |
| When discoverable | Immediately / after moderation | **After photo safety scan** — a scan backlog delays discoverability, but never exposes unsafe content |

### 9.2 FR2 — Get the recommendation stack

**Serve path (fast, every request):**
```
GET /recommendations (A)
  1. LRANGE stack:{A} 0 99                         (Redis, ~1 ms)
  2. If missing, or gen_cell ≠ A.cell (moved), or gen_prefs_version ≠ A.prefs_version
        → synchronous generate (slow path, ~150 ms)
  3. Serve-time filter (all pipelined, ~3 ms):
        BF.MEXISTS swiped_bloom:{A} ids            # already swiped?
        SMISMEMBER recent_swipes:{A} ids           # swiped since Bloom update?
        ZMSCORE served:{A} ids                     # shown in last 72 h?
        blocks:{A}, status cache (banned/hidden/deleted)
  4. Take first 50 survivors; LTRIM consumed; ZADD served:{A}
  5. Hydrate: profile cache MGET → photo URLs → distance = round(dist(A.cell, B.cell))
  6. If LLEN stack:{A} < 75 → produce stack.refill(A)
```

**Generate path (Stack Builder, async or slow path):**
```
  1. shards = geoshards intersecting circle(A.cell, A.max_km)        (usually 1–3)
  2. ES query on those shards only:
       filter: geo_distance(cell_center, A.cell, A.max_km)
               age ∈ [A.age_min, A.age_max]
               gender ∈ A.interested_in
               pref_genders ∋ A.gender                ← reciprocal
               pref_age_min ≤ A.age ≤ pref_age_max    ← reciprocal
               status = ACTIVE, last_active_at > now-30d
       size: 1000, sort: lightweight score (activity recency, distance)
  3. Post-filter: pref_max_km ≥ dist(A,B) (reciprocal distance), Bloom, blocks
  4. Rank (model): P(A likes B) × P(B likes A) × exposure-fairness factor
  5. Write top 200 → stack:{A} (TTL 24 h) with gen_cell, gen_prefs_version
```

**Why reciprocity matters (Staff point):** without filtering on the candidate's preferences, A is shown people who will never see A in their own stack — every like A sends there is guaranteed wasted, which depresses match rate system-wide.

### 9.3 FR3 — Swipe

```
POST /swipes {target=B, decision, client_swipe_id}   by A
  1. Gateway: rate limit (e.g., likes/12 h on free tier, swipes/s anti-bot)
  2. Swipe Svc: lo,hi = sort(A,B); side = (A==lo ? "lo" : "hi")
  3. Pair Store UpdateItem(pk = "lo:hi",
        SET {side}_decision=:d, {side}_ts=:t, {side}_swipe_id=:sid
        CONDITION attribute_not_exists({side}_decision) OR {side}_swipe_id = :sid
        RETURN ALL_NEW)
     - condition fails with different swipe_id → already decided → return current state
  4. If decision=LIKE and other_side_decision=LIKE → Match Svc (FR4)
  5. SADD recent_swipes:{A} B (TTL 24 h)  — immediate dedup before CDC catches up
  6. Respond { matched }
  async: CDC from Pair Store → swipe.recorded → SwipeHistory row, Bloom BF.ADD, ML features
```

**Client side is part of the design:** the UI animates the swipe immediately (optimistic), appends to a local durable queue, and flushes in the background (single or batched). Network latency is invisible except for the match moment.

### 9.4 FR4 — Match and notify

```
Swipe Svc (B's swipe completes the pair)
  └─▶ Match Svc: INSERT match(match_id=hash(lo,hi)) IF NOT EXISTS   + outbox row (same txn)
        ├─ inserted        → { matched:true, match_id } returned to B → "It's a Match!"
        └─ already existed → same response (idempotent retry)
Outbox relay ─▶ Kafka match.created {match_id, lo, hi, completed_by=B}
   ├─▶ UserMatches writer: rows for lo and hi (idempotent upsert on match_id)
   ├─▶ Notification Svc: push to A (the FIRST swiper, who isn't looking at the screen)
   │                     WebSocket event to both if connected; dedupe key = match_id
   └─▶ Chat Svc: create conversation (out of scope)
```

| Decision | Options | Choice |
|---|---|---|
| Who gets a push | Both / first swiper only | **First swiper only** — second swiper already sees the modal; both get list updates |
| Notification semantics | Exactly-once / at-least-once + dedupe | **At-least-once + client/server dedupe on `match_id`** — exactly-once across APNs/FCM isn't achievable end to end |
| Match list storage | Query pair store / separate index | **UserMatches partitioned by `user_id`**, derived from `match.created` (pair store can't answer "all my matches" efficiently) |

---

## 10. Deep Dive 1: Consistency — Exactly One Match, Never a Missed Match

### 10.1 The race

```
Naive design: swipes partitioned by swiper_id. Match check = "did B like A?" on B's partition.

 t0  A swipes right on B ── write (A→B LIKE) to partition(A)
 t0  B swipes right on A ── write (B→A LIKE) to partition(B)
 t1  A's service reads partition(B) for (B→A) … not yet visible → "no match"
 t1  B's service reads partition(A) for (A→B) … not yet visible → "no match"
 ⇒ Both liked each other. No match is ever created. Silent, permanent loss.
```
Read-then-write across two partitions is not atomic. Retrying doesn't help — neither side has anything left to retry. Mutual swipes within a second are rarer than you'd think intuitively but at 23K likes/s they happen continuously, especially when two users are both actively swiping in the same small area.

### 10.2 Options

| Option | How | Pros | Cons |
|---|---|---|---|
| **A. Swiper-partitioned + distributed transaction (2PC)** | Lock both partitions | Keeps natural layout | 2PC on the hottest write path; coordinator failure blocks; latency ×2–3 |
| **B. Swiper-partitioned + serializable DB** (e.g., Spanner / CockroachDB) | Let the DB serialize cross-row txns | Correct, simple code | Cross-shard txns at 23K/s with global consensus latency; cost |
| **C. Pair-key record + single-item atomic update** ✅ | Both swipes write to the same item `lo:hi`; atomic update returns post-image | One key, one atomic op, no locks; second writer *always* sees first | Per-swiper history needs a second, derived table |
| **D. Pair-key + Redis Lua script** | `HSET` own side + read other side atomically in Lua | Sub-ms; trivially atomic (single-threaded per shard) | Redis replication is async — a failover can lose an acknowledged like; needs a durable log behind it |
| **E. Async matcher: Kafka partitioned by pair key** | Single consumer per partition processes swipes serially | No race by construction; very high throughput | Match is seconds late → no instant "It's a Match!"; consumer lag = match lag |
| **F. Cassandra LWT on pair key** | `UPDATE … IF …` (Paxos) | Linearizable per partition in Cassandra | ~4 round trips; LWT throughput limits under heavy use |

**Choice: C as the primary path, E as the degraded mode, plus a reconciler.**

### 10.3 Why the pair key works

```
pair "u_12:u_77"  (u_12 < u_77)
┌────────────┬────────────┬──────────────────┬────────────┬────────────┐
│ lo_decision│ lo_ts      │ lo_swipe_id      │ hi_decision│ hi_ts …    │
└────────────┴────────────┴──────────────────┴────────────┴────────────┘

All updates to one item are serialized by the store (per-key linearizability):
  A writes lo_decision=LIKE → post-image: {lo:LIKE, hi:∅}      → no match
  B writes hi_decision=LIKE → post-image: {lo:LIKE, hi:LIKE}   → MATCH
Whichever write lands second sees both. There is no interleaving where neither does.
```

- **No CAS loop needed:** each user writes only their own attribute, so updates never conflict — the store just orders them.
- **Linearizability scope:** this needs per-key strongly consistent reads of the post-image (DynamoDB single-item update returning `ALL_NEW` from the leader; Cassandra would need LWT, *not* `QUORUM` — replication quorum gives overlap, not a serial order of read-modify-write. Keep the two senses of "quorum" separate; see *CAP/PACELC* guide).
- **Both sides might "see" the match** (e.g., A retries after B's write). That's fine because match creation is **idempotent**: `match_id = hash(lo, hi)`, `INSERT … IF NOT EXISTS`.

### 10.4 Idempotency & retries

| Scenario | Outcome |
|---|---|
| Client retries same LIKE (same `client_swipe_id`) | Condition passes on matching `swipe_id`; same post-image; same response |
| Client sends a *different* decision later for same pair | Condition fails → existing decision stands (Rewind is a separate, explicit paid flow) |
| Match insert times out after succeeding | Retry hits `IF NOT EXISTS` → returns existing match → same `match_id` |
| `match.created` delivered twice | UserMatches upsert by `match_id`; notification dedupe by `match_id` |

### 10.5 Safety nets (Staff: assume your primary path will fail somewhere)

1. **Transactional outbox** for `match.created` — never "write match, then publish" as two steps.
2. **Reconciler:** a stream job on the pair store's CDC flags any item reaching `{LIKE, LIKE}` with no match row after N seconds → creates it idempotently. Catches crashes between step 3 and step 4 of the swipe flow.
3. **Degraded mode:** if the pair store is unavailable, the Swipe Service accepts swipes into a durable Kafka topic keyed by `pair_id` (Option E) and returns `{matched:false, pending:true}`. Matches arrive later by push. We degrade *instant* matching, never *correct* matching, and never refuse the swipe.

---

## 11. Deep Dive 2: Low Latency — Sub-300 ms Recommendation Stacks

### 11.1 Where latency comes from
A naive per-request query is: geo radius + 2 range filters + 2 set filters + 2 *reciprocal* filters + "exclude thousands of swiped IDs" + ranking. That's slow on any engine at 10K/s peak — and dangerously slow at p99.

### 11.2 Options for producing a stack

| Option | How | Pros | Cons |
|---|---|---|---|
| A. Query the primary DB per request (Postgres + PostGIS) | `ST_DWithin` + filters + `NOT IN (swiped)` | Simple; always fresh | Multi-predicate geo queries + huge exclusion lists at 10K/s; read replicas lag; poor p99 |
| B. Search engine per request | ES geo + filters each time | Flexible; fresh | Every stack open pays full query + rank; tail latency from scatter-gather |
| C. Offline batch precompute (nightly Spark) | Compute all stacks daily | Cheap reads | New users invisible for a day; travel breaks it; most precomputed stacks for inactive users wasted |
| **D. Precomputed per-user stack cache + async incremental refill + on-demand fallback** ✅ | Generate ~200 on demand, serve 50 at a time, refill at low-water mark | Serving is O(1); generation off the hot path; fresh within minutes | Cache staleness (moves, pref changes) → explicit invalidation + serve-time filters |

### 11.3 Partitioning the recommendation index — the real Staff question

The index is ~100 GB. *How* you shard it decides which query pays for fan-out.

| Option | Pros | Cons |
|---|---|---|
| Hash-shard by `user_id` | Even data distribution; trivial routing for writes | **Every geo query scatters to all shards**; p99 = slowest of N shards; cluster QPS = N × client QPS |
| Single unsharded index, fully replicated | No fan-out; 100 GB fits in RAM on large nodes | Every location update written to every replica; ceiling as data or update rate grows; one bad mapping change hits all |
| Shard by city / metro | Natural locality | Uneven (NYC vs. rural county); users near boundaries; unmapped areas |
| **Geosharding by grouped cells, balanced by load** ✅ | Query touches only 1–3 shards; shards sized by *users × query rate*, not area | Boundary queries hit multiple shards; users moving across shards need delete+insert; shard map must be rebalanced periodically |

> Tinder engineering has publicly described moving to geosharded recommendation indexes for exactly this reason; the specific mechanics below are illustrative.

```
Geoshard construction (offline, recomputed weekly):
  1. Tile the world in S2/H3 cells (e.g., ~level where a cell ≈ few km²)
  2. Weight each cell by active users × request rate
  3. Greedily merge contiguous cells into shards with ≈ equal weight
     → Manhattan might be several shards; Montana might be one
  4. Publish shard map (cell → shard) to Location Svc, Indexer, Stack Builder

Query routing:
  shards = { shard(cell) for cell in cover(circle(A.cell, A.max_km)) }   # usually 1–3
```

- **Max distance is capped** (e.g., 160 km) so the cover is bounded.
- **Hot events** (festival, NYE in Times Square): add read replicas to that shard; the map rebalances on the next cycle.
- Users who move across shards: Indexer deletes from old shard, inserts into new — ordered by `location_version` so a delayed old update can't resurrect the stale doc.

### 11.4 Keeping the cached stack valid

| Event | Action |
|---|---|
| Preferences changed | Bump `prefs_version` → serve path sees mismatch → regenerate |
| Moved to a new grid cell (or > X km) | Regenerate on next fetch; drop stack |
| Candidate swiped/blocked/banned since generation | Filtered at serve time (Bloom, recent-swipes, status cache) |
| Candidate changed prefs and no longer fits A | Tolerated: small staleness; candidate's next generation corrects it. A blocked/banned candidate is never tolerated (CP check) |
| Stack TTL (24 h) | Regenerate — keeps ranking signals fresh |

**Predictive warm-up:** users have strong daily rhythms; regenerate stacks 15–30 minutes before a user's typical open time so the first fetch of the day is a cache hit.

### 11.5 Latency budget (p99 target < 300 ms; illustrative)

```
Fast path (cache hit, ~95%+ of requests):
  Gateway + auth                          10 ms
  LRANGE stack                             1 ms
  Serve-time filters (pipelined)           3 ms
  Profile hydration MGET (50)              8 ms
  URL/distance construction                1 ms
  Serialization                            5 ms
  ─────────────────────────────────────────────
  p50 ≈ 30 ms, p99 ≈ 80 ms

Slow path (miss / invalidated):
  ES query on 1–3 geoshards               40–80 ms
  Post-filter + Bloom                      5 ms
  Ranking (1,000 candidates, batched)     20–40 ms
  Write stack + serve                     10 ms
  ─────────────────────────────────────────────
  p99 ≈ 200–250 ms
```

### 11.6 Client tricks that make latency invisible
- Fetch the next page when ~20 cards remain; never let the stack hit zero on screen.
- Prefetch photos for the next 3–5 cards; blurhash placeholders render instantly.
- Swipes are optimistic — the card flies away before the server responds.

---

## 12. Deep Dive 3: Scale & Durability — 175K Swipes/s Peak

### 12.1 Pair store choice

| Option | Pros | Cons |
|---|---|---|
| Sharded Postgres (shard by `hash(pair_id)`) | `INSERT … ON CONFLICT DO UPDATE … RETURNING` gives the atomic post-image; familiar | 175K/s needs many shards + manual resharding; vacuum pressure from TTL deletes |
| **DynamoDB (pk=`pair_id`)** ✅ | Single-item atomic update + `ALL_NEW`; native TTL; auto-partitioning; no ops | Per-partition throughput limits (irrelevant here — pair keys are uniformly spread); cost at scale |
| Cassandra | Huge write throughput, TTL, multi-DC | Plain writes aren't read-modify-write atomic; needs LWT for the match check (Paxos cost) |
| Redis (primary) | Fastest | Async replication can lose acked likes → violates durability NFR |

Either DynamoDB or "Cassandra for history + something linearizable for the pair check" are defensible; the point is to state *which property* you need from the store (per-key linearizable read-modify-write) and choose accordingly.

### 12.2 Why the pair key also solves hot spots
A highly attractive profile receives far more likes than average. With **swipee-partitioned** storage, that profile's partition is a hot key. With **pair-key** partitioning, each inbound like goes to a *different* item `(liker, popular_user)` — load spreads uniformly by hash. The anchor constraint pays off twice.

The hot key reappears only in **derived, swipee-centric views** (e.g., a "Likes You" grid or counter):
```
likes_received:  PRIMARY KEY ((swipee_id, bucket), ts, liker_id)
                 bucket = hash(liker_id) % 16 for users above a threshold
counter:         sharded counter, periodically reconciled
```
Same move as the crawler's hot-domain sub-queues and Instagram's celebrity `followers` bucketing.

### 12.3 Write path at scale

```
Client queue ──(batch ≤ 20 swipes / 2 s or on app background)──▶ Gateway
   ▶ Swipe Svc (stateless, autoscaled on QPS)
      ▶ Pair Store: 1 conditional update per swipe (parallel within batch)
      ▶ Redis recent_swipes:{A} SADD (pipelined)
   ◀ ack only after Pair Store write is durable (replicated)
Pair Store CDC ─▶ Kafka swipe.recorded (partition by swiper_id)
   ▶ SwipeHistory writer (Cassandra, partition swiper_id, TTL on PASS)
   ▶ Bloom updater (BF.ADD swiped_bloom:{swiper})
   ▶ ML feature stream / analytics (ClickHouse / warehouse)
```

- **Batching** cuts request overhead at the gateway; each swipe inside the batch is still its own atomic pair update (batching never spans the atomicity boundary).
- **Durability contract:** the client keeps a swipe in its local queue until acked; the server acks only after a durable write. Kafka downstream uses `acks=all` + `min.insync.replicas=2` — `acks` alone isn't the durability contract (see *Kafka* guide).
- **Back-pressure:** if CDC consumers lag, dedup gets slightly stale (covered by `recent_swipes`), matching is unaffected — the match decision never depends on downstream consumers.

### 12.4 Storage lifecycle

| Data | Retention | Why |
|---|---|---|
| Pair records with any LIKE | Long-lived (or until both users deleted) | Needed for matching and for "already liked" semantics |
| Pair records PASS-only | TTL 90 days (product decision) | ~60% of rows; allows intentional resurfacing; caps storage |
| SwipeHistory | Same TTL policy as pair | Derived view |
| Swipe events (analytics) | Hot 30 days, then columnar warehouse | Model training |
| Deleted accounts | Hard-delete pairs/matches via async purge job | Privacy / regulatory deletion |

### 12.5 Abuse & capacity protection
Bots swipe right on everyone to harvest matches for spam. Defenses double as capacity controls: per-user like limits, swipe-velocity anomaly detection, device attestation, and shadow-banning (bot's likes are recorded but never surface in anyone's stack or match).

---

## 13. Deep Dive 4: Never Show a Profile Twice

### 13.1 Sources of repeats

| Source | Example |
|---|---|
| Already swiped | Swiped B last month; B appears again |
| Swiped after stack generated | B is in the cached stack; A swiped B via another device / earlier page |
| Served but not swiped | A closed the app mid-stack; refresh shows the same faces |
| Multi-device | Phone + web served the same candidates |
| Intentional | PASS older than 90 days expired — resurfacing is a product decision, not a bug |

### 13.2 Options for excluding swiped profiles

| Option | Pros | Cons |
|---|---|---|
| ES `must_not terms` with all swiped IDs | Exclusion inside the query | Query payload grows to thousands of IDs; ES terms limits; slow and expensive per request |
| Exact per-user set in Redis (`SMISMEMBER`) | Exact | ~10K IDs × 50M users × Redis set overhead → multiple TB of RAM |
| Exact check against SwipeHistory (Cassandra) | Exact, durable | 1,000-candidate post-filter = large multi-get per generation; p99 risk |
| **Per-user Bloom filter + small exact "recent" set + served-set** ✅ | ~12 KB/user; O(k) per check; exact for the window where Bloom may lag | False positives hide ~1% of never-seen candidates (harmless); no deletes (handled by rebuild) |

**Why a Bloom false positive is acceptable here:** a FP means "skip a candidate A has never seen." With hundreds of candidates in range, losing 1% is invisible. A false *negative* (showing a swiped profile) would be visible — and Bloom filters don't have false negatives. Same asymmetry the *Web Crawler* exploits for URL dedup.

### 13.3 Layered dedup

```
Layer 1 — Generation-time:  Bloom check on ~1,000 ES candidates   (removes bulk)
Layer 2 — Serve-time:       Bloom + recent_swipes:{A} (exact, last 24 h, covers CDC lag)
                            + served:{A} ZSET (shown in last 72 h → skip)
Layer 3 — Client:           session-local seen set (multi-page, multi-tap)
Layer 4 — Swipe Svc:        pair condition rejects a second decision anyway
                            (a repeat that slips through can't corrupt state)
```

### 13.4 Bloom lifecycle

| Concern | Handling |
|---|---|
| Sizing | 10K expected items @ 1% FP → ~96 Kbit (12 KB), 7 hash functions |
| Growth beyond capacity | Scalable Bloom (stacked filters) or rebuild larger from SwipeHistory when fill > threshold (crawler's fill-ratio monitoring) |
| PASS TTL expiry (needs "delete") | Standard Bloom can't delete → periodic rebuild (e.g., monthly) from SwipeHistory, which has already dropped expired passes |
| Cold users (filter evicted from Redis) | Rebuild on next generation from SwipeHistory (one partition scan); serve with exact `recent_swipes` meanwhile |
| Multi-device | Filter is server-side and per user, so all devices share it |

---

## 14. Cross-Cutting: Location Privacy & Safety

**Trilateration attack:** if the API returns exact distance, an attacker spoofs their own location to three points, reads three distances, and solves for the victim's position to within meters. Dating apps have been publicly vulnerable to this. This is the domain's signature correctness trap.

| Mitigation | How |
|---|---|
| Snap before compute | Server computes distance from **grid-cell centers**, never precise coordinates |
| Round output | Whole km/miles; floor at "< 1 km" |
| Never return coordinates | No API echoes location; precise location stored encrypted, readable only by Location Svc |
| Rate-limit location changes | Detect "teleporting" accounts (many large location jumps) |
| Strip photo EXIF | GPS in photo metadata is the other common leak |

**Blocks and bans (CP):** enforced at serve time and in match/chat reads against the source of truth (block cache with write-through), never trusted to the cached stack. Blocking also unmatches and removes both users from each other's stacks immediately.

---

## 15. CAP / PACELC Positioning

| Component | Choice | Reasoning |
|---|---|---|
| **Swipe pair store (match check)** | **CP / EC** (per-key linearizable) | Atomic read-modify-write is the correctness core; on partition, divert to Kafka degraded mode rather than risk a missed match |
| **Match creation** | **CP** (put-if-absent) | Exactly one match row per pair |
| **Blocks / bans** | **CP / EC** | Showing a blocked user is a safety incident |
| Recommendation index (ES) | AP / EL | Slightly stale candidates are fine |
| Stack cache (Redis) | AP / EL | Rebuildable; serve-time filters cover staleness |
| SwipeHistory + Bloom | AP / EL | Lag covered by exact `recent_swipes` set |
| Profile DB | AP-leaning reads (replicas) / CP writes | Profile edits are rare; replica lag of seconds tolerable |
| UserMatches index, notifications | AP, at-least-once + dedupe | Late push is fine; lost push recovered on next app open from match list |
| Photo store / CDN | AP | Immutable content-addressed objects |

Most of the system is AP/EL; the **three named CP exceptions** are the pair-store match check, match creation, and block enforcement.

---

## 16. Failure Modes & Mitigations

| Failure | Symptom | Detection | Mitigation |
|---|---|---|---|
| Simultaneous mutual like | Missed match (silent) | Reconciler finds `{LIKE,LIKE}` w/o match | Pair-key atomic update; reconciler |
| Crash between pair update and match insert | Missed match | Same reconciler | Reconciler + idempotent `match_id` |
| Client retry of LIKE | Duplicate match / double push | Dup `match_id` events | Conditional update on `swipe_id`; put-if-absent; dedupe on `match_id` |
| Pair store partition / outage | Swipes failing | Error rate, latency | Degraded mode: durable Kafka by `pair_id`, async matching; client queue holds unacked swipes |
| Push lost (APNs/FCM) | User unaware of match | Delivery receipts | Match list is source of truth; badge on next open |
| Stack cache node loss | Slow first fetch for a shard of users | Miss-rate spike | Slow-path generation with per-user request coalescing; rate-limit regeneration to protect ES |
| Regeneration stampede after cache loss | ES overloaded | ES queue/latency | Coalesce; prioritize users currently opening the app; serve partial stacks |
| Geoshard hotspot (festival, NYE) | One ES shard saturated | Per-shard QPS/CPU | Add replicas; split shard in next map cycle; widen cache TTL there |
| Stale location (traveled) | Wrong-city candidates | `gen_cell ≠ current cell` | Invalidate on cell change; ordered `location_version` in Indexer |
| Repeated profile | User sees B twice | Client telemetry "repeat seen" | Layered dedup (§13); pair condition makes it harmless |
| Banned/deleted user in cached stacks | Ghost profiles | Status mismatch at hydration | Serve-time status filter; purge event drops from stacks |
| Bot mass-liking | Spam matches; write load | Velocity anomalies | Like limits, attestation, shadow-ban |
| Trilateration | Location leaked | Repeated precise location spoofing | Grid snapping + rounding; teleport detection |
| CDC/Kafka lag | Bloom stale; analytics delayed | Consumer lag | `recent_swipes` exact set; autoscale consumers; match path independent |
| Attractive-profile hot key in "Likes You" | Hot partition | Per-key QPS | Bucketed partition; sharded counters |

**Pattern class:** the popular profile is this domain's version of the crawler's spider trap and Instagram's celebrity — an entity whose degree is orders of magnitude above the mean. Here the primary write path is immune by construction (pair keys), and only derived swipee-centric views need bucketing.

---

## 17. Scalability & Sharding

| Data | Shard key | Why | What becomes expensive |
|---|---|---|---|
| Swipe pair store | `hash(lo:hi)` | Co-locates both halves of a match; uniform spread | "All swipes by A" → derived SwipeHistory |
| SwipeHistory | `swiper_id` | Dedup / Bloom rebuild = one partition | "Who liked me" → separate bucketed table |
| Matches (canonical) | `match_id` | Idempotent creation | "My matches" → UserMatches |
| UserMatches | `user_id` | Match list = one partition | — |
| Profile DB | `user_id` | Point lookups, hydration | Geo queries → search index |
| Recommendation index | **geoshard** (load-balanced cell groups) | Query touches 1–3 shards | Cross-shard moves on travel; rebalancing |
| Stack cache / Bloom / served-set | `user_id` (Redis hash slot) | All per-user ops single-key | — |
| Kafka `swipe.recorded` | `swiper_id` | Ordered per-user history updates | — |
| Kafka degraded-mode matcher | `pair_id` | Serial per-pair processing | — |

**The sharding lesson:** every choice above moves an expensive query somewhere else. Pair-key sharding makes matching cheap and history expensive; geosharding makes stack generation cheap and travel/reindexing expensive. Staff answers name what got expensive and how it's handled.

**Horizontal scaling plan**

| Tier | Lever |
|---|---|
| Stateless services | Autoscale on QPS/CPU |
| Stack Builder, CDC consumers | Autoscale on Kafka lag |
| Pair store | DynamoDB auto-partitioning (or add Postgres/Cassandra shards) |
| ES | Split/replicate hot geoshards; recompute shard map |
| Redis | Add shards; slot migration |

**Multi-region:** users are pinned to a home region by geography (they mostly match locally), so pair writes are almost always single-region. Rare cross-region pairs (travel) route to the region owning the lower `user_id`'s home, keeping the per-key linearizability guarantee.

---

## 18. Senior vs Staff Answer Differentiators

### Senior-level expectations
- Identifies profile, location/geo index, swipe store, match, notification components
- Uses a geospatial index (geohash/quadtree/ES) for nearby users
- Stores swipes and checks for a reverse like on right swipe
- Uses push notifications for matches
- Basic capacity math

### Staff-level expectations (additional)

| Topic | What Staff adds |
|---|---|
| Matching | Raises the **mutual-swipe race unprompted**; pair-key layout so the check is a single-item atomic op; deterministic `match_id`; reconciler; degraded async mode |
| Consistency vocabulary | Linearizable per-key RMW vs. replication quorum; why `QUORUM` ≠ atomic check |
| Recommendations | Reciprocal preference filtering; precomputed stacks with explicit invalidation; two-sided marketplace (exposure fairness) |
| Sharding | "The index is 100 GB — I shard for query fan-out, not size"; geosharding balanced by load |
| Dedup | Layered (Bloom + exact recent set + served-set + client); Bloom FP asymmetry; rebuild for TTL deletes |
| Hot keys | Pair keys make inbound-like hot spots disappear; only derived views need bucketing |
| Privacy | Trilateration attack and grid snapping; blocks as CP |
| Cost | Left-swipe TTL as the dominant storage lever |
| Client | Local durable swipe queue, optimistic UI, prefetch as part of the latency design |

**The single most important Staff move:** recognizing that *a swipe is one user's write but a match is a two-user transaction*, and choosing a layout (pair key) that collapses the transaction to one atomic key — instead of bolting on locks.

---

## 19. Interview Time Allocation

For a 45-minute session:

| Phase | Time | Focus |
|---|---|---|
| Requirements & scope | 4 min | FR1–FR5; park chat/paid features; call out matching consistency + privacy NFRs |
| Capacity estimation | 4 min | 58K/175K swipes/s, 23K match checks/s, 100 GB index, 100 Gbps photos |
| Entities & APIs | 4 min | Pair record, deterministic `match_id`, `client_swipe_id`, sync `matched` |
| High-level design | 6 min | Draw all components; walk FR1–FR4 |
| Matching consistency (DD1) | 9 min | Race, options table, pair key, idempotency, reconciler |
| Stack latency (DD2) | 8 min | Precompute vs on-demand, geosharding, invalidation, reciprocity |
| Swipe scale + dedup (DD3/DD4) | 6 min | Pair-store choice, hot keys, Bloom layering, TTL |
| Failure modes, CAP, privacy | 4 min | Named CP exceptions, trilateration, stampede |

**Cut if short:** photo pipeline, storage lifecycle details. **Never cut:** the mutual-swipe race and pair key, reciprocal filtering, geosharding rationale.

---

## 20. Quick-Reference Cheatsheet

```
KEY NUMBERS (illustrative)
──────────────────────────
50M DAU × 100 swipes          → 5B/day ≈ 58K/s (peak ~175K/s)
40% likes                     → ~23K match checks/s
2% reciprocal                 → ~40M matches/day ≈ 460/s
Stack fetches                 → ~3.5K/s (peak ~10K/s); refills ~1.2K/s internal
Rec index                     → 100M docs × 1 KB ≈ 100 GB (shard for QPS, not size)
Swipe storage                 → ~1 TB/day before RF; TTL left swipes
Bloom                         → ~12 KB/user, ~600 GB for 50M DAU
Photo egress                  → ~104 Gbps avg → CDN

MATCHING
────────
pair_id = min(a,b):max(a,b); each user writes only their own side
Single-item atomic update returning post-image → second writer sees both LIKEs
match_id = hash(lo,hi); INSERT IF NOT EXISTS; outbox → match.created
Reconciler on CDC: {LIKE,LIKE} without match → create
Degraded: Kafka keyed by pair_id, async match, never reject a swipe

RECOMMENDATIONS
───────────────
Precomputed stack:{uid} (~200) in Redis; serve 50; refill at low-water mark
Invalidate on prefs_version or grid-cell change; TTL 24 h; predictive warm-up
ES geosharded by load-balanced cell groups → 1–3 shards per query
Reciprocal filters: my prefs ∋ them AND their prefs ∋ me (age, gender, distance)
Rank: P(A→B) × P(B→A) × exposure fairness

DEDUP
─────
Gen-time Bloom → serve-time Bloom + recent_swipes (exact 24 h) + served (72 h)
→ client seen-set → pair condition as final guard
Bloom FP = skip unseen (harmless); rebuild monthly for TTL'd passes

PRIVACY
───────
Distance from grid-cell centers, rounded; never return coordinates
Trilateration = signature trap; strip EXIF; blocks enforced on read (CP)

CAP
───
CP: pair-store match check, match creation, blocks/bans
AP/EL: rec index, stack cache, swipe history/Bloom, profiles, notifications, CDN

STAFF SIGNALS
─────────────
✓ Swipe = 1-user write; match = 2-user txn → pair key collapses it
✓ Linearizable RMW ≠ replication quorum
✓ Index is small → geoshard for fan-out, not capacity
✓ Reciprocal preference filtering
✓ Layered dedup with Bloom FP asymmetry
✓ Popular-profile hot key vanishes on write path; bucket derived views
✓ Trilateration + grid snapping
✓ Left-swipe TTL as storage lever
```

---

## 21. Related Guides

| Guide | Relationship |
|---|---|
| *Proximity Service* | Geohash/quadtree/S2 fundamentals behind the geosharded index |
| *Yelp* | Geo search with multi-predicate filters; ES geo queries |
| *Messaging Application (WhatsApp/MS Teams style)* | Chat after match |
| *Notification System* | Match push delivery, dedupe, APNs/FCM |
| *Instagram* | Photo upload control/data plane split, renditions, CDN, blurhash |
| *News Feed System* | Precomputed per-user lists (timelines ↔ stacks) and invalidation |
| *Web Crawler* | Bloom filter sizing, FP asymmetry, fill-ratio rotation; spider trap ↔ popular profile pattern class |
| *Distributed Unique ID Generation* | Snowflake user IDs; deterministic vs. random IDs for idempotency |
| *Sharding Strategies* | Pair-key and geo sharding; "which query becomes expensive" |
| *CAP / PACELC Theorem* | Per-component positioning; linearizable RMW vs. replication quorum |
| *Distributed Key-Value Store (Dynamo-style)* | Pair store semantics, conditional writes, TTL |
| *Apache Kafka* | Outbox/CDC, `acks` + `min.insync.replicas`, degraded-mode matcher |
| *Leaderboard System* | Desirability/score bucketing analogies for ranking |

---

*Guide compiled for Senior & Staff-level system design interview preparation.*
*Topics: Tinder, Geosharded Recommendations, Mutual-Swipe Atomicity, Bloom Filter Dedup, Location Privacy.*
