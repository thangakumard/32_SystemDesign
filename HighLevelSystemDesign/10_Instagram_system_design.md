# Instagram — System Design Interview Guide
### Senior & Staff Engineer Level

---

## Table of Contents

1. [Problem Statement & Scope](#1-problem-statement--scope)
2. [Requirements](#2-requirements)
3. [Capacity Estimation](#3-capacity-estimation)
4. [High-Level Architecture](#4-high-level-architecture)
5. [Component 1: Media Upload & Processing Pipeline](#5-component-1-media-upload--processing-pipeline)
6. [Component 2: Post Metadata Service & ID Generation](#6-component-2-post-metadata-service--id-generation)
7. [Component 3: Social Graph Service](#7-component-3-social-graph-service)
8. [Component 4: Feed Generation — Fan-out Strategy](#8-component-4-feed-generation--fan-out-strategy)
9. [Component 5: Feed Ranking & Assembly](#9-component-5-feed-ranking--assembly)
10. [Component 6: Stories (Ephemeral Content)](#10-component-6-stories-ephemeral-content)
11. [Component 7: Explore / Discovery Page](#11-component-7-explore--discovery-page)
12. [Component 8: Engagement — Likes, Comments, Counters](#12-component-8-engagement--likes-comments-counters)
13. [Component 9: Media Storage & CDN Delivery](#13-component-9-media-storage--cdn-delivery)
14. [Component 10: Notifications & Direct Messages](#14-component-10-notifications--direct-messages)
15. [Component 11: Content Moderation & Trust/Safety](#15-component-11-content-moderation--trustsafety)
16. [CAP / PACELC Positioning](#16-cap--pacelc-positioning)
17. [Failure Modes & Mitigations](#17-failure-modes--mitigations)
18. [Scalability & Sharding](#18-scalability--sharding)
19. [Senior vs Staff Answer Differentiators](#19-senior-vs-staff-answer-differentiators)
20. [Interview Time Allocation](#20-interview-time-allocation)
21. [Quick-Reference Cheatsheet](#21-quick-reference-cheatsheet)

---

## 1. Problem Statement & Scope

Design **Instagram**: a media-first social network where users post photos and videos, follow other users, view a ranked feed of content from accounts they follow, post ephemeral **Stories** (24-hour expiry), discover new content via an **Explore** page, and engage via likes, comments, and direct messages.

### Anchor constraint (the thing everything else derives from)

Instagram is fundamentally **a media-delivery problem wearing a social-graph costume**. Twitter's anchor constraint is fan-out under a power-law follower distribution; Instagram inherits that same constraint *and* adds a second, orthogonal one: **every post carries large, immutable binary payloads (images/video) that must be processed into multiple renditions and served at extremely high read:write ratios through a CDN.** Every component below traces back to one of these two constraints — the social fan-out problem, or the media pipeline problem — and the components that get this design wrong usually collapse the two into one pipeline instead of treating them as separately-scaled systems.

### What the interviewer is really testing

- Do you separate the **metadata/fan-out path** (small, hot, needs low latency) from the **media path** (large, cold after first view, needs a CDN and tiered storage)?
- Can you correctly reuse fan-out and ranking patterns from adjacent systems (Twitter, generic News Feed) rather than re-deriving them, while calling out where Instagram's are genuinely different (ephemeral Stories, non-social Explore ranking)?
- Do you understand the **hybrid fan-out / celebrity problem** as it applies here — and its media-specific twist (a celebrity's photo also has to be transcoded and CDN-warmed before the fan-out even completes)?
- Do you treat **content moderation and privacy (blocking, private accounts) as first-class, not bolted on**?

### Relationship to this library's other guides

This guide is written to sit alongside, not duplicate, three existing guides:

| Existing guide | What it already owns | What this guide does instead |
|---|---|---|
| Twitter / Fan-out Architecture | Push/pull/hybrid fan-out mechanics, the celebrity/hot-key problem, timeline read path | References the mechanics directly; adds the media-pipeline coupling that makes Instagram's celebrity problem strictly harder than Twitter's |
| News Feed System (general ranked feed) | ML ranking pipeline, feature store, ad serving vs. billing split, block-list CP/EC exception | References the ranking pipeline directly; focuses on what's upstream (candidate generation sources unique to Instagram: Stories, Explore) rather than the ranker internals |
| YouTube | Video transcoding, adaptive bitrate (HLS/DASH), CDN tiering | References the transcoding pipeline directly for Instagram Reels/video; this guide covers the *photo* pipeline in full since it has no prior home in the library |

At Staff level, explicitly naming this division of scope in the interview — "the fan-out mechanics here are the same hybrid push/pull pattern as Twitter's, so let me focus on what's different" — is itself a signal. It shows you're not pattern-matching from a blank slate; you're composing known solutions and reasoning about the deltas.

---

## 2. Requirements

### Functional Requirements

| Requirement | Notes |
|---|---|
| Upload photo/video posts with caption, tags, location | Immutable once posted (edits are metadata-only: caption, not pixels) |
| Follow / unfollow users; public and private accounts | Private accounts require an accepted follow request |
| View a ranked home feed from followed accounts | Not strictly chronological — ML-ranked |
| Post & view Stories (24-hour auto-expiring content) | Separate surface from the feed; sequential "ring" viewing UX |
| Explore / Discovery page | Content from accounts you don't follow, ranked by predicted interest |
| Like, comment, save posts | Real-time-ish counters; idempotent on double-tap |
| Direct messages (1:1 and group) | Out of primary scope; covered briefly in §14 |
| Block / restrict users; report content | Must be enforced consistently across feed, Explore, search, comments |
| Push notifications for engagement | Likes, comments, new followers, DMs |

### Non-Functional Requirements

| Requirement | Target |
|---|---|
| Scale | Hundreds of millions of DAU; illustrative, not pinned |
| Read:write ratio | Very high — feed/story reads dominate writes by ~1000:1, same order as Twitter |
| Feed read latency | p99 < 200ms for feed assembly (excluding media fetch, which is CDN-served separately) |
| Media upload → visible in feed | Seconds, not minutes, for non-celebrity accounts |
| Availability | Feed/Explore/Stories: prioritize availability (AP) — a stale or slightly incomplete feed beats an error page |
| Durability | Uploaded media must never be lost once acknowledged to the client |
| Consistency exceptions | Block-lists, private-account visibility, and content-moderation takedowns must be strongly consistent — availability is *not* the priority there |

---

## 3. Capacity Estimation

All figures below are illustrative order-of-magnitude reasoning tools, not facts to defend under follow-up pressure — the interviewer cares that you can derive and stress-test them, not that you've memorized Instagram's actual numbers.

```
Assume: 500M DAU (order of magnitude)

POST WRITES
  ~100M posts/day (photo + video combined)
  100,000,000 / 86,400 ≈ 1,160 posts/sec average, 5-10x at peak

FEED READS
  Avg 20 feed refreshes/user/day → 500M × 20 = 10B feed reads/day
  10,000,000,000 / 86,400 ≈ 115,700 reads/sec average

STORY READS  (higher engagement, lower payload per item)
  Avg 5 story-ring opens/user/day, ~5 stories/ring viewed
  500M × 5 × 5 = 12.5B story views/day → ~145,000 views/sec average
  → Confirms: story traffic can exceed feed traffic in raw QPS,
    but with a much smaller and TTL-bounded working set (24h)

MEDIA STORAGE (raw, before multi-rendition fan-out)
  Assume post mix: 80% photo (avg 2MB original), 20% video (avg 20MB)
  80M photos × 2MB  = 160 TB/day
  20M videos × 20MB = 400 TB/day
  ≈ 560 TB/day raw uploads

MULTI-RENDITION MULTIPLIER (the number interviewers under-estimate)
  Each photo needs ~5-6 derived renditions (thumbnail, feed, profile-grid,
  full-res, WebP/AVIF variants) → effective stored bytes ≈ 1.3-1.5x raw
  Each video needs multiple bitrate/resolution renditions for adaptive
  streaming (see YouTube guide) → effective stored bytes ≈ 2-3x raw

BANDWIDTH (the dominant cost, same conclusion as the Web Crawler guide
  reached for storage — here it's egress, not ingress)
  If each post is viewed ~1,000 times on average (power-law skewed) and
  a CDN-delivered feed rendition is ~150KB:
  100M posts × 1,000 views × 150KB ≈ 15 PB/day of egress
  → CDN cost, not origin storage, dominates the bill. This is the
    single most important capacity insight for this system.

CELEBRITY SPIKE
  A single post from a 300M-follower account, if even 5% view it in the
  first hour: 15M reads/hour ≈ 4,200 reads/sec sustained for ONE post —
  larger than the entire average system throughput for many services.
  This is why fan-out and CDN warming for high-follower accounts cannot
  share an unbounded queue with normal accounts (see §8, §13).
```

**Key insight (mirrors the Web Crawler guide's storage insight, inverted):** for the crawler, storage was the dominant cost because content is *written* once at huge scale. For Instagram, **egress bandwidth** is the dominant cost because content is *read* thousands of times more often than it's written. Every architectural choice in the media path (§5, §13) is downstream of minimizing origin-fetch rate via aggressive CDN caching.

---

## 4. High-Level Architecture

```
                         ┌──────────────┐
                         │    Client    │
                         └──────┬───────┘
                                │
                    ┌───────────▼────────────┐
                    │   API Gateway / LB      │
                    └───┬─────────────────┬───┘
                        │                 │
          ┌─────────────▼───┐   ┌─────────▼──────────┐
          │  WRITE PATH      │   │  READ PATH          │
          └─────────────┬───┘   └─────────┬────────────┘
                        │                 │
    ┌───────────────────▼───┐   ┌─────────▼─────────────────┐
    │ Media Upload Service   │   │ Feed Service               │
    │ (pre-signed URL issue) │   │ (fan-out read/hybrid)      │
    └──────────┬─────────────┘   │ + Ranking Service          │
               │                 │ + Stories Service           │
       ┌───────▼────────┐        │ + Explore Service            │
       │ Object Store    │        └───────┬────────────────────┘
       │ (raw upload)    │                │
       └───────┬────────┘                │
               │ event                    │
       ┌───────▼─────────────┐            │
       │ Kafka: media-uploaded │           │
       └───────┬─────────────┘            │
               │                          │
    ┌──────────▼───────────┐              │
    │ Media Processing Fleet │             │
    │ resize · transcode ·   │             │
    │ thumbnail · moderation │             │
    │ scan (NSFW/CSAM/etc.)  │             │
    └──────────┬───────────┘              │
               │                          │
       ┌───────▼────────┐         ┌───────▼────────┐
       │  CDN Origin     │◀────────│  Post Metadata  │
       │  (multi-rendition)│       │  Service        │
       └───────┬────────┘         │  + Social Graph │
               │                  └───────┬────────┘
       ┌───────▼────────┐                 │
       │      CDN        │        ┌────────▼────────┐
       │ (edge, global)  │        │ Kafka: post-created│
       └───────┬────────┘        └────────┬────────┘
               │                          │
               ▼                 ┌────────▼─────────────┐
            Client               │ Fan-out Worker Fleet   │
                                  │ (push to follower       │
                                  │  feed caches — hybrid)  │
                                  └──────────────────────┘
```

The diagram deliberately splits into a **write path** (upload → process → fan-out) and a **read path** (feed/Stories/Explore assembly), because they scale on entirely different axes: write path scales with posts/sec and is CPU/GPU-bound (transcoding); read path scales with DAU × sessions and is I/O/cache-bound. Conflating them into one service is a Senior-level mistake this guide explicitly avoids.

---

## 5. Component 1: Media Upload & Processing Pipeline

### Upload Flow

```
1. Client requests a pre-signed upload URL from Media Upload Service
2. Client uploads directly to Object Store (S3/GCS) — bypasses app
   servers entirely, avoiding a bandwidth bottleneck on the API tier
3. Object Store fires an event (S3 event notification / GCS Pub/Sub)
4. Event lands on Kafka topic: media-uploaded
5. Media Processing Fleet consumes, performs:
     a. Virus/malware scan on the raw file
     b. EXIF stripping (privacy — GPS coordinates, device serial numbers)
     c. Content moderation scan (NSFW, violence, CSAM hash-matching
        against known-bad hash databases — see §15)
     d. Multi-rendition generation (photo) or transcoding (video)
     e. Perceptual hash computation (duplicate/reupload detection,
        also feeds moderation)
6. On success → Post Metadata Service is notified → post becomes visible
7. On moderation failure → post is quarantined, never reaches fan-out
```

**Why pre-signed direct-to-object-store upload, not through the app tier?** At 1,160 posts/sec average with multi-MB payloads, routing raw bytes through stateless app servers wastes bandwidth and compute on pure pass-through. This is the same "keep large payloads off the hot compute path" principle used in the Web Crawler guide's decision to stream fetch responses directly rather than buffer them in the router.

### Multi-Rendition Generation (Photo)

```
From one uploaded original, generate in parallel:
  • Thumbnail       (~150×150,  for grid views)
  • Feed rendition  (~1080px wide, primary feed display)
  • Story rendition (~1080×1920, full-bleed vertical)
  • Profile-grid     (~320×320)
  • Format variants: WebP/AVIF for supporting clients, JPEG fallback

Each rendition stored separately in the object store, keyed by
{post_id}/{rendition}.{format}, and pushed to CDN origin.
```

**Staff-level point:** rendition generation must be **idempotent and retry-safe**. A worker crash mid-generation must not leave a post half-rendered (e.g., thumbnail exists, feed rendition doesn't) — that half-state is worse than a fully-failed post, because it can pass a naive "does the post exist" check while breaking the client UI. Use a manifest record that's only marked complete after *all* renditions succeed; the feed/Stories services check the manifest, not individual rendition existence.

### Video Handling

Video posts (including Reels) are handed to the same transcoding architecture described in this library's **YouTube guide**: adaptive-bitrate output (HLS/DASH), multiple resolution/bitrate ladders, and a separate, more expensive processing fleet than photo resizing. The key delta from YouTube: Instagram video is short-form and upload-to-visible latency matters far more than for long-form YouTube uploads, so the transcoding queue for Instagram video needs a tighter SLA and often trades some encoding efficiency for speed on the first-available rendition (serve a lower-quality rendition immediately, backfill higher quality asynchronously).

### Upload Idempotency (adversarial/correctness concern)

A client retry on a flaky connection must not create a duplicate post. Fix: client generates an **idempotency key** (UUID) at capture time, included in the upload request; Media Upload Service deduplicates on `(user_id, idempotency_key)` for a bounded window (e.g., 24h) before creating a new post record.

---

## 6. Component 2: Post Metadata Service & ID Generation

### What It Stores

```sql
CREATE TABLE posts (
  post_id       BIGINT PRIMARY KEY,   -- Snowflake-style, see below
  user_id       BIGINT,
  caption       TEXT,
  location_id   BIGINT NULL,
  media_type    ENUM('photo','video','carousel'),
  media_refs    JSON,                 -- rendition keys, not raw bytes
  created_at    TIMESTAMP,
  visibility    ENUM('public','followers','private'),
  moderation_status ENUM('pending','approved','removed')
) PARTITION BY HASH(user_id);
```

### ID Generation

Post IDs use a **Snowflake-style generator** — same scheme covered in this library's **Distributed Unique ID Generation** guide: 41-bit timestamp, machine/worker-ID bits, sequence bits. Two properties matter specifically for Instagram:

- **Time-sortability**: a post's ID roughly orders by creation time, which lets Stories (strictly chronological within a user's ring) and any "most recent" fallback ranking use the ID directly instead of a separate `ORDER BY created_at` index scan.
- **No coordination on the hot path**: matches the write-path anchor constraint — post creation must not block on a synchronous consensus round.

Refer to the Unique ID Generation guide for worker-ID assignment (ZooKeeper ephemeral znodes) and clock-drift handling; this guide doesn't re-derive them.

### Why Metadata Is Separated From Media

The metadata row is small (bytes), hot, and needs low-latency point lookups and range scans by user. The media is large (megabytes) and CDN-served. Storing media references (not bytes) in the metadata store — same principle as the Web Crawler guide's separation of the URL metadata store (Cassandra) from raw HTML (S3) — keeps the metadata store's working set small enough to stay largely in cache.

---

## 7. Component 3: Social Graph Service

Stores the follow relationships and drives both the fan-out target list (§8) and visibility checks (private accounts, blocks).

```sql
-- Two tables, deliberately denormalized for opposite query directions
CREATE TABLE followers (follower_id BIGINT, followee_id BIGINT, since TIMESTAMP,
  PRIMARY KEY (followee_id, follower_id));   -- "who follows me" — fan-out source
CREATE TABLE following (follower_id BIGINT, followee_id BIGINT, since TIMESTAMP,
  PRIMARY KEY (follower_id, followee_id));   -- "who do I follow" — feed pull source
```

Storing both directions (rather than one table queried both ways) trades write amplification (2 writes per follow) for read-path simplicity — the same "own the write cost so reads stay O(1)" tradeoff this library's guides consistently favor for read-heavy systems.

### Private Accounts & Blocking

A follow to a private account creates a `pending` request row, not an active edge — the edge is only materialized on acceptance. Blocking removes the edge bidirectionally and inserts a row into a **block-list** that every downstream visibility check (feed, Explore, search, comments, DMs) must consult.

**This is the one component in the entire system that should not default to AP.** Consistent with this library's News Feed guide, block-list checks and private-account visibility are flagged **CP/EC** — a stale block-list read that briefly shows a blocked user's content is a trust-and-safety failure, not a minor UX blemish, so this path accepts latency cost for consistency, against the grain of everything else in this design.

---

## 8. Component 4: Feed Generation — Fan-out Strategy

This is the same **hybrid push/pull fan-out** mechanics as this library's Twitter guide — full derivation (front/back queues analog, precompute vs. on-demand tradeoffs, the celebrity/hot-key problem) lives there. Summarized for reference:

```
Normal accounts (< ~10K-100K followers, threshold tunable):
  → Fan-out on write (push): on post creation, a worker fleet writes
    the post_id into every follower's precomputed feed cache (Redis
    list, capped length e.g. 800 entries). Feed reads are then O(1).

Celebrity accounts (above threshold):
  → Fan-out on read (pull): the post is NOT pushed to millions of
    caches. Instead, the feed-read path merges in celebrity posts
    on demand at request time (a small "who do I follow that's a
    celebrity" list, typically single-digit-to-low-hundreds per user).

Hybrid merge at read time:
  final_feed = merge(precomputed_push_feed, on_demand_celebrity_posts)
             → re-ranked by Feed Ranking Service (§9)
```

### The Instagram-Specific Twist: Media Readiness Gates Fan-out

Twitter's celebrity problem is purely a fan-out/write-amplification problem — a tweet is text, ready to fan out the instant it's written. Instagram's is strictly harder, because **fan-out cannot start until media processing (§5) completes.** A celebrity's photo posted to 300M followers means:

1. Media processing must finish (renditions generated, moderation passed) — this is now on the critical path to feed visibility, not just storage.
2. CDN origin must be warmed with the popular renditions *before* the fan-out (or pull-time merge) exposes the post, or the first wave of viewers all miss-hit the CDN edge simultaneously and hammer the origin — a self-inflicted thundering herd.

**Staff-level mitigation:** for accounts above the celebrity threshold, the post-creation pipeline inserts an explicit **CDN pre-warm step** between "media processing complete" and "post becomes visible" — pushing the most common renditions (feed, thumbnail) to a set of edge PoPs proactively, sized by the account's historical view velocity. This adds a few hundred milliseconds of visibility latency for a tiny population of accounts in exchange for protecting the origin from a correlated spike — a direct architectural echo of the Web Crawler guide's Bloom-filter-fill-ratio-triggered rotation: pay a small, controlled cost proactively rather than an uncontrolled cost reactively.

---

## 9. Component 5: Feed Ranking & Assembly

Ranking pipeline internals (feature store, ML model serving, multi-source candidate blending, ad insertion) are owned by this library's **News Feed System** guide and aren't re-derived here. What's specific to Instagram at this layer:

### Candidate Sources Feeding the Ranker

```
1. Push feed cache (followed accounts, non-celebrity)     — §8
2. On-demand celebrity pull                                — §8
3. "Suggested posts" from Explore's candidate generator     — §11
4. Ads (owned by News Feed guide's ad-serving split)
```

Instagram's ranker blends candidate source (4) into what's nominally a "following" feed — the single biggest Senior-vs-Staff tell on this component is whether a candidate correctly identifies that **the modern Instagram feed is not purely social-graph-sourced**; treating it as "just show posts from people I follow, ranked" is an incomplete model that misses where a meaningful fraction of engagement actually originates.

### Ranking Signals Specific to Instagram

| Signal | Why it matters here specifically |
|---|---|
| Watch-time / completion rate on video | Reels-style content rewards completion, not just impression |
| Time since posted, decayed non-linearly | Freshness matters more for Stories-adjacent content than static posts |
| Media type diversity in a session | Avoid an all-video or all-carousel feed feeling repetitive |
| Close-friends / interaction-recency graph weight | Denser recent interaction (comments, DMs) outweighs raw follow-time |

---

## 10. Component 6: Stories (Ephemeral Content)

Stories are architecturally distinct enough from the permanent feed to warrant their own service, not a `type` flag on the Post table.

### Why a Separate System, Not a Post Variant

| Dimension | Feed Post | Story |
|---|---|---|
| Lifetime | Permanent (until deleted) | Fixed 24h TTL |
| Read pattern | Ranked, non-sequential | Strictly chronological, ring-based (per-author sequence) |
| Storage tier | Durable, tiered (hot→cold) | TTL-bounded — can use a store with native expiry |
| "Seen" tracking | Not typically per-viewer at this granularity | Every viewer's watch position is tracked, for the seen/unseen ring indicator |

### Design

```
Story metadata: Redis with TTL = 24h, key = story_id, auto-expires —
  no cleanup job needed, unlike the feed's soft-delete + reaper pattern.

Story ring assembly (per viewer):
  1. Fetch list of followed accounts with an active (non-expired) story
     — a simple Redis SET per followee, checked via the social graph
  2. Order rings by: unseen-first, then recency/interaction-weight
  3. Within a ring: strict chronological order of that author's stories

Seen-tracking:
  viewer_id + story_id → watched, stored in a wide, TTL-matched table
  (same 24h expiry — no need to retain seen-state past the story's life)
```

**Staff-level point:** because Stories auto-expire, this is one of the few places in the whole system where you get **storage lifecycle management for free** from the data store's native TTL support, instead of building a background reaper (contrast with the Web Crawler guide's explicit retention-policy background jobs for raw HTML). Calling this out — recognizing where a data store's built-in feature eliminates an entire class of operational code — is a concrete Staff-level signal.

### Fan-out for Stories

Stories use **pure push fan-out on creation**, even for celebrity accounts, because the total addressable content per author is small and short-lived (a handful of story items per day, gone in 24h) — the celebrity/hot-key problem from §8 doesn't reproduce here the same way, since there's no large accumulating feed cache to write into, just a bounded "has an active story" flag per followee that's cheap to fan out at any follower count.

---

## 11. Component 7: Explore / Discovery Page

Explore surfaces content from accounts the viewer does **not** follow — the social graph is largely irrelevant here; this is a recommendation-systems problem, not a fan-out problem.

### Candidate Generation

```
Multiple parallel candidate generators, blended:
  1. Collaborative filtering — "users who engaged with X also engaged
     with Y" (co-engagement embeddings)
  2. Content-based similarity — visual/caption embedding nearest-neighbor
     to posts the viewer has engaged with
  3. Trending/velocity-based — posts with unusually high engagement
     velocity relative to the poster's historical baseline
  4. Account-affinity expansion — accounts similar to ones already followed

Each generator produces a candidate pool (e.g., 500 posts); a
lightweight scorer merges and truncates before handing off to the
same-family ranking model referenced from the News Feed guide.
```

### Why This Cannot Reuse the Feed's Push Fan-out

Feed fan-out (§8) precomputes a bounded per-user list because the *candidate set* (accounts you follow) is bounded and known in advance. Explore's candidate set is effectively "all public content on the platform," which cannot be precomputed per-user at write time — it must be generated at or near read time from an index (approximate nearest-neighbor search over embeddings, e.g., FAISS/ScaNN-style vector index), refreshed on a much looser cadence (minutes, not seconds) than the feed.

**CAP/PACELC framing:** Explore is the most AP/EL surface in the entire system — staleness of minutes is invisible to the user, and a degraded or partially-populated Explore grid is a far better failure mode than an error page, so this is where the system leans hardest into availability over freshness.

---

## 12. Component 8: Engagement — Likes, Comments, Counters

### The Counter Problem

A like/unlike is a small write, but on a viral post it can arrive at extreme concurrency — the same **hot-key problem** as the Web Crawler guide's hot-domain issue and the Twitter guide's celebrity fan-out, manifesting here as a **hot database row** instead of a hot queue.

```
Naive: UPDATE posts SET like_count = like_count + 1 WHERE post_id = X
  → single row becomes a write hot-spot; lock contention collapses
    throughput exactly when the post is most popular (worst possible
    time for it to fail)

Fix: Sharded counters
  like_count_shard(post_id, shard_id) → each increment picks a random
  shard (0-N); true count = SUM across shards, computed lazily/cached
  and refreshed periodically rather than read-after-every-write

Fix: Async aggregation via Kafka
  Like events are appended to Kafka topic: post-likes (not a direct
  DB write). A stream aggregator batches and flushes count deltas
  periodically. Trades read-after-write consistency on the exact
  count for write availability under extreme concurrency.
```

**PACELC framing:** likes are **PA/EL** — available under partition, and even absent a partition the system chooses lower latency over strict consistency (an eventually-accurate like count, refreshed within seconds, is entirely acceptable; nobody audits their like count in real time).

### Idempotency (adversarial/correctness concern)

A double-tap or a client retry must not double-count. The like action is modeled as a **set membership operation**, not an increment: `(user_id, post_id) → liked_at`. The displayed counter is derived from aggregate deltas (above), but the *has this user liked this post* check is a direct, idempotent lookup — inserting the same `(user_id, post_id)` pair twice is a no-op, not a double-increment.

### Comments

Comments are lower-volume than likes but need ordering (chronological or ranked-by-relevance thread) and are subject to the same content-moderation pipeline as posts (§15) — comment spam/abuse volume on a viral post can itself become a hot-key write problem, mitigated the same way (Kafka-buffered writes, sharded storage keyed by `post_id`).

---

## 13. Component 9: Media Storage & CDN Delivery

### Storage Tiers

| Tier | What | System | Rationale |
|---|---|---|---|
| Origin (durable) | All renditions, all posts | Object store (S3/GCS) | Source of truth; CDN can always refetch |
| CDN edge | Hot/recently-viewed renditions | CDN PoPs (global) | Absorbs the overwhelming majority of read traffic (§3: ~15PB/day egress) |
| Cold archive | Renditions for posts with near-zero view velocity after N months | Glacier/Coldline-equivalent | Same retention-tiering principle as the Web Crawler guide's raw-HTML lifecycle |

### Cache Key Design

```
CDN cache key: {post_id}/{rendition}/{format}
  — deliberately excludes any per-viewer state (no user_id, no session)
  so that a single cached object serves every viewer requesting that
  rendition, maximizing cache hit ratio. Any per-viewer overlay
  (e.g., "liked by X" annotations) is applied client-side or via a
  separate low-weight metadata call, never by cache-busting the image.
```

**Why this matters at Staff level:** the single most common mistake that silently destroys CDN hit ratio is leaking per-user or per-session data into a cache key that should be identical for every viewer of the same content. This is the same principle as the Web Crawler guide's URL-normalization-before-dedup step — normalize/minimize the key before it hits the cache layer, or the cache layer can't do its job.

### Adaptive Image Delivery

Client requests a rendition based on device pixel density and network conditions (similar in spirit to HLS/DASH's adaptive bitrate ladder for video, covered in the YouTube guide, but for static images): a `srcset`-equivalent negotiation lets a slow-network client fall back to a lower-resolution rendition without a separate round trip, since all renditions were pre-generated at upload time (§5) rather than computed on demand.

---

## 14. Component 10: Notifications & Direct Messages

### Notifications

Push notifications (new follower, like, comment, mention) reuse this library's **Notification System** guide's architecture directly — fan-out-on-event via Kafka, per-channel delivery (push/in-app/email), user-preference filtering, and dedup/batching logic (e.g., "Alice and 40 others liked your photo" batching, to avoid a notification storm from a viral post — the same hot-key concern as §12, applied to the notification fan-out path instead of the counter).

### Direct Messages (brief — out of primary scope)

DMs are effectively a separate real-time messaging system (bidirectional, low-latency, often WebSocket/long-poll-based) with its own delivery-guarantee and read-receipt concerns, closer in shape to a chat system design problem than to the feed/media problem this guide focuses on. Worth naming explicitly in an interview as a system boundary — "DMs deserve their own design, I'll treat them as an external dependency here" — rather than either ignoring them or trying to force them through the feed's fan-out machinery, which is the wrong tool for point-to-point low-latency delivery.

---

## 15. Component 11: Content Moderation & Trust/Safety

Treated as first-class, not an afterthought — consistent with this library's principle (see the VirusTotal-style scanning guide) that adversarial concerns belong in the initial design, not bolted on after a follow-up question.

```
Moderation touchpoints across the system:
  1. Upload time (§5): NSFW classifier, CSAM hash-matching against
     known-bad perceptual hash databases, spam/scam visual pattern
     detection — gates whether a post ever reaches fan-out at all
  2. Post-publish (reactive): user reports feed into a review queue;
     confirmed violations trigger takedown, which must propagate to
     the CDN (cache invalidation) and all fan-out caches
  3. Comment/DM abuse: text classifiers on comments, rate limits on
     new/low-trust accounts to blunt spam bursts
  4. Block-list enforcement (§7): must be checked at feed assembly,
     Explore candidate generation, search, and comments — a single
     missed enforcement point is a trust failure, not a bug ticket
```

**Takedown propagation is a correctness problem, not just a moderation problem.** Once a post is removed, it must disappear from: the origin store (or be flagged unservable), every CDN edge that cached it, every precomputed feed cache it was pushed into (§8), and Explore's candidate index (§11). This is architecturally the same "invalidate everywhere it was copied" problem as the Web Crawler guide's Bloom-filter-rotation and dedup-hash consistency concerns — the fix is the same pattern: an event (`post-removed`) on Kafka that every downstream consumer (CDN invalidation, feed-cache purge, Explore index) subscribes to, rather than a point-to-point call from the moderation service to each system individually.

---

## 16. CAP / PACELC Positioning

| Component | CAP / PACELC | Reasoning |
|---|---|---|
| Media upload → object store | **AP / EL** | Uploads are append-only; availability preferred, eventual propagation to CDN is fine |
| Post metadata store | **AP**, tunable per-query | `QUORUM` reads for a user's own just-posted content (so they see it immediately); `ONE` for others' feed reads |
| Social graph — follow edges | **AP / EL** | A brief staleness in follower count is invisible; write availability matters more |
| Social graph — block-list / private-account visibility | **CP / EC** | The one deliberate exception. Correctness of who-can-see-what outweighs latency here |
| Feed cache (push fan-out) | **AP / EL** | Missing the newest post for a few seconds is fine; an unavailable feed is not |
| Celebrity pull-time merge | **AP / EL** | Same reasoning; also bounded by the small "celebrities I follow" list, so recompute cost is low |
| Stories store | **AP / EL** | TTL-bounded, low stakes; availability strongly preferred over strict ordering across replicas |
| Explore index | **AP / EL** (loosest in the system) | Minutes of staleness invisible to users; recommendation quality doesn't require strong consistency |
| Like/comment counters | **AP / EL** | Sharded, eventually-aggregated; exact real-time accuracy is explicitly not a goal |
| Like/comment *membership* (has user X liked post Y) | **CP-leaning** for the idempotency check itself, though the aggregate count remains AP | Correctness of "did I already like this" matters for UX even though the displayed count doesn't need to be exact |
| CDN edge cache | **AP / EL** | Origin is the consistency source of truth; edges are intentionally stale within a bounded TTL |
| Content moderation takedown propagation | **CP-leaning / EC** | A takedown must win the race against continued serving; this is the system's second deliberate consistency-over-latency exception, alongside block-lists |
| Coordination (Snowflake worker-ID assignment, sharding rebalance) | **CP** | Same as every other guide in this library — coordination decisions favor consistency, and are kept off the hot request path |

**The pattern worth stating explicitly in an interview:** this system is overwhelmingly AP/EL, with exactly two deliberate, named exceptions — **privacy/blocking** and **moderation takedowns** — where the system trades away latency and availability for consistency. Stating the default *and* naming the exceptions explicitly (rather than either blanket-labeling everything AP or agonizing over every component individually) is the efficient way to demonstrate this at Staff level within interview time constraints.

---

## 17. Failure Modes & Mitigations

| Failure | Symptom | Detection | Mitigation |
|---|---|---|---|
| Celebrity post fan-out storm | Fan-out worker queue backs up; feed-cache writes lag | Kafka consumer lag on `post-created` topic | Hybrid fan-out (§8): celebrities use pull-time merge, not push |
| CDN thundering herd on new viral post | Origin store overwhelmed by correlated cache misses | Origin request-rate spike correlated with a single post_id | Pre-warm CDN edges before visibility (§8); origin request coalescing |
| Half-rendered media (crash mid-processing) | Post visible with missing renditions; broken client UI | Manifest completeness check fails | Idempotent, retry-safe rendition pipeline; manifest gates visibility (§5) |
| Hot-row like counter contention | Write latency spikes on viral posts | DB lock-wait time on `posts` row | Sharded counters + async Kafka aggregation (§12) |
| Notification storm from viral post | Push provider rate-limited; delivery backlog | Notification queue depth | Batch/dedup notifications ("X and 40 others liked...") |
| Stale block-list read exposes blocked content | User sees content from someone who blocked them | User report / audit | Block-list is the deliberate CP exception (§7, §16); no caching of this check |
| Takedown doesn't propagate to a fan-out cache | Removed content still appears in some users' feeds | Post-removal audit sampling | `post-removed` event fanned out via Kafka to every consumer that copied the content (§15) |
| Explore index staleness after a takedown | Removed content still surfaces in recommendations | Same as above | Explore consumes the same `post-removed` event; loosest SLA of any consumer, but still subscribed |
| Snowflake clock skew on ID-generation nodes | Duplicate or out-of-order post IDs | Same failure mode as this library's Unique ID guide | Same mitigation: refuse-or-wait on clock regression, NTP slewing |
| Regional CDN PoP outage | Elevated latency/errors for users in that region | CDN provider health metrics | Multi-CDN or multi-PoP failover; origin absorbs overflow temporarily |
| Media processing fleet backlog | Upload-to-visible latency spikes | Kafka consumer lag on `media-uploaded` topic | Autoscale processing fleet; degrade gracefully (serve lower-res rendition first, backfill later) |

---

## 18. Scalability & Sharding

Sharding strategy for the metadata and social-graph stores follows the general principles in this library's **Sharding Strategies** guide directly; the Instagram-specific choices are:

| Store | Shard key | Why |
|---|---|---|
| Post metadata | `user_id` (or `hash(user_id)`) | Most queries are "this user's posts"; keeps a user's content collocated |
| Social graph (followers/following) | `user_id` on each directional table | Matches the two dominant query patterns directly (§7) |
| Like/comment counters | `hash(post_id, shard_id)` | Deliberately spreads a single hot post's writes across shards (§12) |
| Feed cache (Redis) | `hash(user_id)` | Feed reads are always per-viewer; no cross-user query ever needed |
| Media object store | Managed (S3/GCS auto-scales) | No manual sharding needed; key design (§13) is the only lever that matters |
| Explore vector index | Sharded by embedding-space partition (e.g., IVF/product-quantization cluster) | Approximate nearest-neighbor search doesn't need global exactness; sharding by cluster keeps per-query fan-out bounded |

### Horizontal Scaling Summary

```
Media processing fleet    → autoscale on Kafka consumer lag (media-uploaded)
Fan-out worker fleet      → autoscale on Kafka consumer lag (post-created)
Feed/Explore/Stories APIs → stateless, scale on request rate
CDN                        → managed, scales automatically; the real lever is
                              cache-key hygiene (§13), not node count
Post metadata store        → shard by user_id, add shards, resize via
                              consistent hashing (same approach as the
                              Web Crawler guide's Cassandra partitioning)
```

---

## 19. Senior vs Staff Answer Differentiators

### Senior-level expectations

- Correctly identify the major components (upload, feed, social graph, storage, CDN)
- Describe basic push/pull/hybrid fan-out for the feed
- Mention a CDN for media delivery
- Handle obvious failure modes (upload failure, post not found)
- Provide basic capacity math

### Staff-level expectations (additional)

| Topic | What Staff-level adds |
|---|---|
| System framing | Names the two orthogonal anchor constraints (social fan-out + media pipeline) up front, and explicitly scopes against adjacent guides instead of re-deriving them |
| Fan-out | Identifies the Instagram-specific twist: media readiness gates fan-out, and CDN pre-warming is required before celebrity content is exposed |
| Ranking | Recognizes the modern feed is not purely social-graph-sourced — Explore/suggested-post candidates are blended in |
| Stories | Justifies a separate service (not a Post variant) on read-pattern and TTL-lifecycle grounds; notes free lifecycle management from native TTL support |
| Explore | Correctly frames it as a recommendation-systems problem, not a fan-out problem — candidate generation via embeddings/ANN search, not precomputed lists |
| Counters | Proactively raises the hot-row problem on viral posts and proposes sharded counters + async aggregation before being asked |
| CDN | Calls out cache-key hygiene (excluding per-viewer state) as the single highest-leverage lever for hit ratio |
| Moderation | Treats takedown propagation as a correctness/consistency problem (event-driven invalidation across every consumer), not just a policy problem |
| CAP/PACELC | States the AP/EL default once, then names the exactly two deliberate consistency-over-availability exceptions (privacy/blocking, moderation takedowns) rather than re-deriving CAP position per component from scratch |
| Failure modes | Proactively surfaces celebrity-post-specific failures (fan-out storm, CDN thundering herd) rather than only generic ones |

### The single most important Staff differentiator

**Recognizing that Instagram is two coupled but separately-scaled systems — a social fan-out problem and a media delivery problem — and that the hardest failures live at their seams** (media readiness gating fan-out; a takedown needing to propagate across both the metadata fan-out caches *and* the CDN). A Senior-level answer treats media as a storage detail bolted onto a feed system. A Staff-level answer treats the coupling between the two as the central design challenge.

---

## 20. Interview Time Allocation

For a 45-minute system design session:

| Phase | Time | Focus |
|---|---|---|
| Requirements & scope | 5 min | Confirm functional + non-functional; state the two anchor constraints |
| Capacity estimation | 5 min | Posts/sec, feed reads/sec, egress bandwidth as the dominant cost |
| High-level architecture | 5 min | Draw write path / read path split; name Kafka as the backbone |
| Media upload & processing (deep dive) | 8 min | Direct-to-object-store upload, rendition generation, idempotency, moderation gate |
| Feed fan-out & the celebrity problem (deep dive) | 10 min | Hybrid push/pull; explicitly reference Twitter guide mechanics, add media-readiness twist |
| Stories & Explore | 5 min | Justify Stories as a separate system; frame Explore as recommendation, not fan-out |
| Engagement counters | 3 min | Hot-row problem, sharded counters, idempotent like semantics |
| CAP/PACELC + failure modes | 4 min | State AP/EL default, name the two CP exceptions, hit 2-3 proactive failure modes |

**What to cut if short on time:** Direct messages (name it as an explicit out-of-scope boundary), notification batching detail. **Never cut:** the fan-out/celebrity-problem deep dive, or the CAP exceptions — these are where the interview signal concentrates.

---

## 21. Quick-Reference Cheatsheet

```
ANCHOR CONSTRAINTS
──────────────────
1. Social fan-out under a power-law follower distribution (same as Twitter)
2. Media pipeline: large immutable binary payloads, huge read:write skew

KEY NUMBERS (illustrative, not pinned)
───────────────────────────────────────
500M DAU  →  ~1,160 posts/sec avg  →  ~115,700 feed reads/sec avg
~560 TB/day raw media uploads  →  ~15 PB/day CDN egress (the real cost)
Celebrity post: up to ~4,200 reads/sec sustained for ONE post

KEY ALGORITHMS / PATTERNS
──────────────────────────
Fan-out:        Hybrid push (normal) / pull (celebrity) — see Twitter guide
Post ID:        Snowflake-style — see Unique ID Generation guide
Ranking:        Multi-source ML ranker — see News Feed guide
Video pipeline: Adaptive bitrate transcoding — see YouTube guide
Counters:       Sharded + async Kafka aggregation, not direct row increment
Cache keys:     {post_id}/{rendition}/{format} — no per-viewer state

CAP / PACELC DEFAULT + EXCEPTIONS
───────────────────────────────────
Default:     AP / EL everywhere (feed, Stories, Explore, counters, CDN)
Exception 1: Block-list / private-account visibility → CP / EC
Exception 2: Moderation takedown propagation → CP-leaning / EC

STAFF-LEVEL SIGNALS TO HIT
───────────────────────────
✓ Name both anchor constraints (social fan-out + media pipeline) up front
✓ Explicitly scope against Twitter/News-Feed/YouTube guides instead of
  re-deriving their mechanics
✓ Media readiness gates fan-out for celebrity posts; CDN pre-warm step
✓ Stories as a separate service, justified on read-pattern + TTL grounds
✓ Explore framed as recommendation/ANN search, not fan-out
✓ Sharded counters + idempotent like semantics, proactively
✓ Cache-key hygiene as the highest-leverage CDN lever
✓ Takedown propagation as an event-driven invalidation problem
✓ AP/EL default stated once, with exactly two named CP exceptions
✓ Proactive celebrity-specific failure modes (fan-out storm, CDN herd)
```

---

*Guide compiled for Senior & Staff-level system design interview preparation.*
*Topics: Instagram, Media Pipeline, Fan-out, Feed Ranking, Stories, Explore, CAP/PACELC Theorem.*
*Companion guides: Twitter/Fan-out Architecture (fan-out mechanics, celebrity problem), News Feed System (ranking pipeline, ad serving), YouTube (video transcoding, adaptive bitrate), Distributed Unique ID Generation (Snowflake post IDs), Notification System (push delivery), Sharding Strategies, Apache Kafka, CAP/PACELC Theorem.*
