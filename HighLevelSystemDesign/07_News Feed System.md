# News Feed System — System Design Interview Guide
### Senior & Staff Engineer Level

---

## Table of Contents

1. [Problem Statement & Scope](#1-problem-statement--scope)
2. [Requirements](#2-requirements)
3. [Capacity Estimation](#3-capacity-estimation)
4. [High-Level Architecture](#4-high-level-architecture)
5. [Component 1: Post Creation Service](#5-component-1-post-creation-service)
6. [Component 2: Social Graph Service](#6-component-2-social-graph-service)
7. [Component 3: Fan-out Service (Push vs Pull vs Hybrid)](#7-component-3-fan-out-service-push-vs-pull-vs-hybrid)
8. [Component 4: Feed Store (Precomputed Timeline Cache)](#8-component-4-feed-store-precomputed-timeline-cache)
9. [Component 5: Ranking Service (Candidate Generation + ML Scoring)](#9-component-5-ranking-service-candidate-generation--ml-scoring)
10. [Component 6: Feed Aggregation & Assembly Service](#10-component-6-feed-aggregation--assembly-service)
11. [Component 7: Media Storage & CDN](#11-component-7-media-storage--cdn)
12. [Component 8: Ad Insertion & Content Mixing](#12-component-8-ad-insertion--content-mixing)
13. [The Celebrity Problem (Fan-out Asymmetry)](#13-the-celebrity-problem-fan-out-asymmetry)
14. [CAP / PACELC Positioning](#14-cap--pacelc-positioning)
15. [Failure Modes & Mitigations](#15-failure-modes--mitigations)
16. [Scalability & Sharding](#16-scalability--sharding)
17. [Feed Freshness & Real-Time Signal Pipeline](#17-feed-freshness--real-time-signal-pipeline)
18. [Senior vs Staff Answer Differentiators](#18-senior-vs-staff-answer-differentiators)
19. [Interview Time Allocation](#19-interview-time-allocation)
20. [Cross-References to Other Guides in This Library](#20-cross-references-to-other-guides-in-this-library)
21. [Quick-Reference Cheatsheet](#21-quick-reference-cheatsheet)

---

## 1. Problem Statement & Scope

A **news feed system** generates a personalized, continuously-scrolling stream of content — posts, photos, videos, links — from the people, pages, and groups a user follows, ranked by predicted relevance rather than pure recency, and blended with ads and suggested content. Canonical examples: Facebook News Feed, Instagram Feed, LinkedIn Feed, X/Twitter Home Timeline.

### What the interviewer is really testing

- Can you reason about the **fundamental fan-out tradeoff** (push vs. pull vs. hybrid) and its cost curve as follower count grows non-uniformly?
- Do you treat **ranking as a first-class distributed system**, not a black box — candidate generation, feature serving, model inference, all with latency budgets?
- Can you identify and solve the **celebrity / hot-key problem**, the domain-specific "spider trap" of this system?
- Do you make **explicit CAP/PACELC decisions per component**, including the subtle case where a privacy concern (blocked users) forces you to trade latency for consistency against the grain of the rest of the system?
- Can you separate **serving-path availability** from **ledger-path consistency** (ads billing, engagement counts) without conflating the two?

---

## 2. Requirements

### Functional Requirements

| Requirement | Notes |
|---|---|
| Create posts (text, photo, video, link) | Author-side write path |
| Follow / friend other users, pages, groups | Directed or bidirectional graph edges |
| View a ranked, paginated feed of followed content | Core read path |
| Like, comment, share | These become ranking signals, not just UI actions |
| Infinite scroll with stable pagination | Feed must not duplicate/drop items as new posts arrive mid-scroll |
| "New posts available" indicator | Soft real-time freshness signal without full push |
| Respect privacy: blocked users, post visibility (public/friends/custom) | Correctness-critical, not just a filter |
| Mix in ads and suggested ("you may like") content | Separate systems feeding into the same assembly layer |

### Non-Functional Requirements

| Requirement | Target |
|---|---|
| Scale | ~500M DAU, ~100M posts/day |
| Read latency | p99 feed load < 200ms |
| Write latency | Post visible to *some* followers within seconds (push path) |
| Availability | Feed must always render *something* — stale beats empty |
| Read:Write ratio | Extremely read-heavy (~100:1) |
| Consistency | Eventual for content/ranking; must be stronger for privacy/visibility checks |

---

## 3. Capacity Estimation

```
Assumptions: 500M DAU, avg 200 follows/user (heavy-tailed), 20% of DAU post ≥1x/day

Posts:
  500M × 0.2 ≈ 100,000,000 posts/day
  100M / 86,400 ≈ 1,157 posts/sec  (peak 3× ≈ 3,470/sec)

Feed reads:
  500M DAU × ~20 feed requests/day (sessions + pagination)
  ≈ 10,000,000,000 reads/day
  10B / 86,400 ≈ 115,740 reads/sec  (peak 3× ≈ 347,000/sec)

Read : Write ratio ≈ 10B : 100M ≈ 100 : 1
  → this ratio is the single number that justifies precomputing
    (fan-out-on-write) rather than computing every feed from scratch.

Naive fan-out cost (push to every follower, no hybrid):
  1,157 posts/sec × 200 avg followers ≈ 231,400 feed-cache writes/sec
  (this ignores celebrity skew — see Section 13)

Feed cache footprint (Redis):
  500M DAU × cap of 800 post_ids × 16 bytes (post_id + score)
  ≈ 500M × 12.8KB ≈ 6.4 TB across the Redis cluster

Post metadata (Cassandra, excludes media):
  100M posts/day × ~1KB ≈ 100 GB/day metadata

Media storage:
  ~40% of posts carry media, avg 300KB compressed
  40M × 300KB ≈ 12 TB/day → object store + CDN
```

**Key insight:** the 100:1 read:write ratio is what makes this a *fan-out-on-write* problem in the common case — you pay a write-amplification cost once per post so that reads are O(1) cache lookups. The entire architecture, and its main failure mode (the celebrity problem), falls out of this one ratio.

As with any interview estimate: these numbers are directionally illustrative, not facts to defend under pressure. The *shape* of the math (reads dominate writes by ~2 orders of magnitude) is the point, not the exact digit.

---

## 4. High-Level Architecture

```
                         WRITE PATH (Post Creation)
                         ──────────────────────────
                    ┌──────────────┐
                    │   Client     │
                    └──────┬───────┘
                           │  POST /post
                    ┌──────▼───────┐
                    │ API Gateway  │
                    └──────┬───────┘
                           │
                    ┌──────▼────────────┐
                    │ Post Creation Svc │──────▶ Media Object Store (S3) + CDN
                    │ (assigns post_id) │
                    └──────┬────────────┘
                           │ writes
                    ┌──────▼───────┐
                    │  Post Store  │  (Cassandra, partition by author_id)
                    └──────┬───────┘
                           │ publish event
                    ┌──────▼───────────────┐
                    │ Kafka: post-events   │  (partitioned by author_id)
                    └──────┬───────────────┘
                           │
                    ┌──────▼───────────────┐
                    │   Fan-out Service    │──────▶ Social Graph Service
                    │  (push / hybrid)     │        (who follows this author?)
                    └──────┬───────────────┘
                           │ fan-out writes (non-celebrity authors)
                    ┌──────▼───────────────┐
                    │  Feed Store (Redis)  │  one sorted-set per follower
                    └───────────────────────┘


                         READ PATH (Feed Fetch)
                         ───────────────────────
                    ┌──────────────┐
                    │   Client     │
                    └──────┬───────┘
                           │  GET /feed?cursor=...
                    ┌──────▼───────┐
                    │ API Gateway  │
                    └──────┬───────┘
                           │
                    ┌──────▼────────────────────┐
                    │ Feed Aggregation Service   │
                    └───┬────────┬────────┬──────┘
                        │        │        │
           ┌────────────▼──┐ ┌───▼──────┐ ┌▼──────────────────┐
           │ Feed Store     │ │ Pull     │ │ Ads / Suggested    │
           │ (Redis —       │ │ Fan-out  │ │ Content Service    │
           │ push-fanned    │ │ (reads   │ │                    │
           │ posts)         │ │ celebs   │ │                    │
           │                │ │ live)    │ │                    │
           └────────┬───────┘ └───┬──────┘ └─────────┬──────────┘
                     │             │                  │
                     └──────┬──────┴─────────┬────────┘
                            │  candidate post_ids
                     ┌──────▼────────────────┐
                     │   Ranking Service      │◀──── Feature Store
                     │  (candidate scoring)   │      (online + offline)
                     └──────┬─────────────────┘
                            │ ranked post_ids
                     ┌──────▼────────────────┐
                     │ Hydration Service      │  (Post Store, media
                     │                        │   CDN URLs, privacy filter)
                     └──────┬─────────────────┘
                            │
                     ┌──────▼───────┐
                     │    Client    │
                     └──────────────┘
```

**Kafka** (`post-events`) decouples post creation from fan-out exactly the way it decouples fetching from parsing in a crawler — it lets fan-out fall behind under load without losing posts, and lets you replay events to rebuild feed caches after an incident.

---

## 5. Component 1: Post Creation Service

### What it does

Validates and persists a new post, assigns it a globally unique, roughly time-sortable `post_id`, kicks off media processing, and emits an event that the rest of the system reacts to asynchronously.

### Write Pipeline

```
1. Validate content (length, spam/abuse filters, rate limits per author)
2. If media present: issue pre-signed upload URL, client uploads directly
   to object store (bypasses app servers for large payloads)
3. Assign post_id (Snowflake-style: time-ordered, globally unique,
   no coordination — see Distributed Unique ID Generation guide)
4. Write post to Post Store (Cassandra, partitioned by author_id)
5. Publish event to Kafka topic: post-events
   { post_id, author_id, timestamp, visibility, media_refs }
6. Return post_id to client immediately (don't block on fan-out)
```

### Why author_id partitioning (not post_id hash)

The pull path (Section 7) needs "give me author X's last N posts" to be a cheap, single-partition scan. Hashing by `post_id` would scatter one author's posts across the whole cluster, turning every pull-model read into a scatter-gather. Partition by `author_id`; use `post_id` only as the clustering key within that partition.

### Staff-Level Note

Returning success to the client **before** fan-out completes is the correct choice, not a shortcut. The author doesn't need their post visible to 50M followers before their own client renders "posted." Decoupling perceived-latency from fan-out-completion-latency is precisely what the Kafka boundary buys you.

**Time to spend in interview:** ~3 minutes. State the pipeline, mention async fan-out, pivot to the graph or fan-out service.

---

## 6. Component 2: Social Graph Service

### What it is

Stores follow/friend edges and answers two very different query shapes:

| Query | Used by | Shape |
|---|---|---|
| "Who follows author X?" | Fan-out service (push) | Potentially millions of rows — needs pagination/streaming |
| "Who does viewer Y follow?" | Pull path, feed aggregation | Bounded (~hundreds), low latency |

### Design

```
Forward index (followees):  user_id → [followed_user_ids]
Inverse index (followers):  user_id → [follower_user_ids]

Both indexes maintained on every follow/unfollow — write-time
denormalization, because the read patterns above are incompatible
with a single normalized edge table at this scale.
```

Sharded by `user_id` hash (both indexes), heavily read-replicated — the follow graph is read orders of magnitude more than it's written (every post's fan-out triggers a followers-list read; only a tiny fraction of activity is the follow/unfollow edge write itself).

### Staff-Level Gotcha

**Follower-list read for a celebrity is itself a scalability problem** — reading 50M follower IDs to fan out one post shouldn't be a single blocking call. Stream it in pages/batches to the fan-out workers rather than materializing the full list in memory. This is the same principle as the crawler's cold-tier promotion job: never require the whole dataset in memory to make progress.

---

## 7. Component 3: Fan-out Service (Push vs Pull vs Hybrid)

This is **the heart of the system** — the equivalent of the URL Frontier in a crawler. Almost every downstream tradeoff traces back to the choice made here.

### Push Model (fan-out-on-write)

```
On new post:
  for each follower F of author:
    ZADD feed:{F} <rank_score> <post_id>
```

| Pros | Cons |
|---|---|
| Read is O(1) — just read the precomputed sorted set | Write-amplifies by follower count |
| Consistent low read latency regardless of follow count | Wasted work for followers who never log in |
| Works even if followee's Post Store shard is unavailable | Catastrophic for celebrities (Section 13) |

### Pull Model (fan-out-on-read)

```
On feed request for viewer V:
  followees = SocialGraph.get_followees(V)          # ~200
  candidates = []
  for F in followees:
     candidates += PostStore.get_recent_posts(F, limit=10)
  merge_and_sort(candidates)
```

| Pros | Cons |
|---|---|
| Zero wasted writes — only computed when actually requested | Read fans out to hundreds of sources per request |
| Cheap for authors with huge follower counts | Read latency scales with number of followees |
| No storage cost for precomputed feeds | Repeated work for the same popular followee across many viewers |

### Hybrid Model (what production systems actually run)

```
if author.follower_count < CELEBRITY_THRESHOLD (~10K):
    push fan-out to all followers' feed stores      # cheap, bounded
else:
    do NOT push — mark as "pull-required" author
    at read time, feed aggregation service pulls this
    author's recent posts live and merges them in
```

```
Read path becomes:
  candidates = FeedStore.get(viewer_id)              # push-fanned posts
             + PullFanout.get_recent(viewer.celeb_followees)  # small list
  merge, rank, return
```

**Why this works:** the vast majority of authors have few enough followers that push is cheap, and the vast majority of *follows* (for any given viewer) are of non-celebrities, so the pull-side list per viewer stays small (viewers typically follow only a handful of true celebrities, even if that handful accounts for a huge share of total edges system-wide).

### Senior vs Staff on this component

**Senior-level answer:** "We'd use a hybrid of push and pull." Correct, but incomplete.

**Staff-level answer:** names the threshold mechanism, explains *why* the threshold makes both sides of the hybrid cheap simultaneously (bounded push fan-out AND bounded pull-list-per-viewer), and proactively raises what happens at the threshold boundary — see Section 13.

---

## 8. Component 4: Feed Store (Precomputed Timeline Cache)

### Structure

```
Redis sorted set per viewer:
  key:    feed:{user_id}
  member: post_id
  score:  rank_score (initially timestamp; re-ranked at read time)

Capped length: trim to top 800 entries on every write
  ZREMRANGEBYRANK feed:{user_id} 0 -801
```

### Design Decisions

| Decision | Reasoning |
|---|---|
| Cap at ~800 posts | Users rarely scroll past a few hundred; unbounded growth wastes memory for no read benefit |
| Store `post_id` only, not full content | Keeps entries tiny (~16 bytes); content hydrated on read from Post Store — separates hot small state from cold large state |
| Skip fan-out writes for inactive users (>30 days) | Mirror image of the crawler's "minimum crawl guarantee" — here the risk is the opposite: wasted work for users who'll never read it. Mitigation: lazily rebuild via pull-on-demand when they return |
| TTL-free but LRU-evictable tier | Reactivating a dormant user triggers cache-miss → rebuild via pull fallback (Section 7) rather than paying to keep every dormant feed warm indefinitely |

### Staff-Level Gotcha: Cache Miss on Rebuild

If a user's feed cache is evicted or lost (node failure, cold start), don't fail the read — fall back to the pull model to reconstruct a feed live, then asynchronously warm the cache in the background. This graceful-degradation path is what makes "availability over consistency" (Section 14) actually true in practice rather than just a table entry.

---

## 9. Component 5: Ranking Service (Candidate Generation + ML Scoring)

Reverse-chronological ordering is the Senior-level answer. A **learned ranking function** is the Staff-level expectation for a "News Feed" (as distinct from a purely chronological timeline).

### Two-Stage Pipeline

```
Stage 1: Candidate Generation (cheap, high recall)
  Pull a few hundred candidates from:
    - Feed Store (push-fanned posts)
    - Pull fan-out (celebrity followees)
    - Group/page posts
    - Suggested content service (explore/exploit)
  No expensive scoring yet — just gather everything plausible.

Stage 2: Scoring (expensive, high precision)
  For each candidate, compute a relevance score using a model that
  blends predicted engagement probabilities:

  score = w1·P(like) + w2·P(comment) + w3·P(share)
        + w4·P(hide/report)  [negative weight]
        + w5·affinity(viewer, author)
        + w6·recency_decay(post_age)
        + w7·content_quality_signal

  Latency budget: ~50ms for a few hundred candidates.
```

### Feature Store: Online + Offline Split

| Feature type | Examples | Freshness | Computed by |
|---|---|---|---|
| Offline / batch | Viewer-author affinity, long-run engagement rate | Hours–1 day | Nightly Spark/Flink batch job |
| Online / real-time | "Liked in last 5 min," trending velocity | Seconds | Kafka → Flink → online KV store |

A ranking request reads **both** — a batch-computed affinity score and a streaming freshness signal — merged at inference time. Missing either one degrades ranking quality but shouldn't fail the request (default to neutral feature values on a feature-store miss).

### Staff-Level Nuances

- **Explore/exploit:** always ranking purely by predicted engagement creates a feedback loop (filter bubble, engagement-bait amplification). Reserve a small slice of slots for exploration/diversity-boosted content, and for content from under-engaged-with sources, to avoid the ranking model training on its own biased output.
- **Business-rule re-ranking pass:** after ML scoring, apply hard constraints — content-type diversity (don't show 5 photos from the same author back-to-back), ad-slot cadence, freshness floor (don't let a 3-day-old high-scoring post permanently bury same-day content).
- **Graceful degradation ladder:** if the model server is slow or down, fall back to a cheaper heuristic ranker (recency + basic affinity), not to a failed request. Cross-reference the Circuit Breaker Pattern guide — the ranking call site is exactly the shape that pattern is for.

---

## 10. Component 6: Feed Aggregation & Assembly Service

### Responsibilities

```
1. Gather candidates: Feed Store + pull fan-out + groups/pages + ads
2. Deduplicate (a post can arrive via multiple paths, e.g. shared post)
3. Privacy/visibility filter — see below
4. Call Ranking Service for final order
5. Hydrate: batch-fetch post content, author metadata, media CDN URLs
6. Paginate with a stable cursor
7. Return to client
```

### Cursor-Based Pagination (solves "feed drift")

Offset-based pagination (`page=2`) breaks when new posts arrive between requests — items shift, causing duplicates or skips. Instead, encode the cursor as `(rank_score, post_id)` of the last item returned:

```
cursor = base64({ last_score: 0.842, last_post_id: "9f3a..." })

Next page query: "give me items ranked below this (score, id) pair"
```

This makes pagination stable relative to a point in the ranking, immune to new inserts above that point.

### The Privacy Correctness Trap (Staff-level differentiator)

This is this system's version of the crawler's "spider trap" or the finance guide's "halt exclusion" — a domain-specific correctness edge that separates a good design from a great one.

```
Before returning any hydrated post, check:
  - Is the viewer blocked by the author, or vice versa?
  - Does the post's visibility setting (public/friends/custom-list)
    include the viewer?

This check must be against FRESH state, not a stale cache — showing
a blocked user's content, or content to someone just removed from a
custom audience, is a trust and correctness failure, not a
performance nuisance.
```

Concretely: this is one place in an otherwise AP-leaning system where it's correct to accept extra latency for a synchronous, consistent read against the block/visibility store, rather than trusting a possibly-stale replica. See Section 14.

---

## 11. Component 7: Media Storage & CDN

- Client uploads media directly to object storage via a pre-signed URL (bypasses app servers for large payloads).
- Post stores media **references** (IDs/URLs), never raw bytes — keeps Post Store rows small and keeps the Feed Store entries tiny.
- Images: multiple resolutions generated async (thumbnail, feed-size, full-res); video: transcoding pipeline is out of scope for this guide's depth — see the YouTube Video Streaming guide for transcoding ladder, adaptive bitrate, and CDN edge-caching details, which apply unchanged to video posts in a feed.
- CDN serves all media reads; the Feed/Post Store path is never on the hot path for actual media bytes.

---

## 12. Component 8: Ad Insertion & Content Mixing

### Two Separate Concerns — Do Not Conflate Them

| Concern | Consistency need | System |
|---|---|---|
| Ad **serving** (which ad shows, in what slot) | AP — availability matters more than perfect targeting on any single request | Real-time low-latency service, merged into feed at a fixed cadence (e.g., 1 ad per 5 organic posts) |
| Ad **billing/accounting** (impression counted, advertiser charged) | CP — must not double-charge or lose an impression | Separate ledger system, strongly consistent, decoupled from the serving path via an async event log |

**Staff-level signal:** explicitly separating "what the user sees" (fast, best-effort) from "what the advertiser is billed for" (slow, exactly-once-ish, reconciled) is the kind of distinction that shows systems maturity — conflating them into one consistency model is a common Senior-level gap.

Suggested/recommended content ("Pages you may like") follows the same explore/exploit logic as ranking (Section 9) and is mixed in by the aggregation service alongside organic and ad candidates.

---

## 13. The Celebrity Problem (Fan-out Asymmetry)

This is the domain-specific "spider trap" of a news feed system — the failure mode most likely to be the deep-dive of the interview, and the one a Staff candidate should raise **before being asked**.

### The Math

```
Regular user posts:  ~200 followers → 200 fan-out writes. Trivial.

Celebrity posts:     50,000,000 followers → 50M fan-out writes
                      for a SINGLE post event.

At the naive push rate, one celebrity post can momentarily dwarf the
entire system's steady-state write volume (Section 3: ~231K/sec
steady-state naive fan-out from ALL posts combined).
```

### Mitigations (in order of typical adoption)

| Mitigation | Mechanism |
|---|---|
| **Hybrid push/pull threshold** | Above N followers, stop pushing; pull live at read time (Section 7) |
| **Partial fan-out + pull backfill** | Push only to the most-active fraction of followers immediately; backfill the rest asynchronously, lower priority |
| **Priority fan-out queue** | Fan out to currently-active/online users first (they'll notice immediately); batch the long tail |
| **Fan-out worker parallelism per post** | Shard a single celebrity post's follower list across many workers rather than one worker iterating millions of IDs serially |
| **Request coalescing on read** | If a viral celebrity post causes a read stampede on the same post_id in the Post Store, coalesce concurrent identical reads into one backing fetch, serve from a shared in-flight cache |
| **Rate limiting fan-out throughput** | Cap fan-out workers' write rate to the Feed Store cluster so a single celebrity event can't saturate Redis and starve unrelated writes |

### Staff-Level Framing

Notice the shape: this is structurally the same problem as the crawler's "hot domain monopoly" (Section 15 of that guide) and "spider trap" — **one entity generating disproportionate, bursty load that a uniform design doesn't anticipate.** Naming that pattern-level connection explicitly in an interview is itself a strong Staff signal — it shows the failure mode is recognized as a *class* of problem, not a one-off fact memorized about feed systems.

---

## 14. CAP / PACELC Positioning

| Component | CAP | PACELC (Else) | Reasoning |
|---|---|---|---|
| Social Graph (read path) | **AP** | EL (favor Latency) | Stale follower/following list for a few seconds is harmless |
| Feed Store (Redis) | **AP** | EL | Stale feed beats no feed; availability is the product requirement |
| Post Store (Cassandra) | **AP** | EL | Tunable; `QUORUM` reads where freshness matters, `ONE` for throughput |
| Fan-out queue (Kafka) | **AP** | EL | At-least-once delivery; idempotent fan-out consumers dedupe by post_id |
| Feature Store (ranking) | **AP** | EL | Missing/stale feature degrades ranking quality, not correctness |
| Ranking model serving | **AP** | EL | Fallback to heuristic ranker on failure (Section 9) beats failing the request |
| **Privacy / block-list check** | **CP** | EC (favor Consistency) | Showing content to a blocked viewer is a trust/legal failure, not a quality degradation — worth the extra latency of a consistent read |
| Ad serving | **AP** | EL | Best-effort targeting; a suboptimal ad beats a blank slot |
| **Ad billing ledger** | **CP** | EC | Must not double-charge or drop an impression — decoupled from serving via async reconciliation |
| Crawl-style coordination (shard assignment, if used for fan-out worker partitioning) | **CP** | EC | Coordinator decisions must be consistent; use ZooKeeper/etcd |

**The one deliberately contrarian row is privacy/block-list.** Every other component in this system leans AP/EL because a feed is inherently a best-effort, eventually-consistent product surface. Privacy is the exception, and calling that out explicitly — rather than blanket-labeling the whole system AP — is exactly the "domain-specific correctness trap" pattern this library treats as the Staff differentiator (see also: halt-exclusion in finance, tap-to-pay zero-network in payments).

---

## 15. Failure Modes & Mitigations

| Failure | Symptom | Detection | Mitigation |
|---|---|---|---|
| Celebrity post fan-out storm | Feed Store write latency spikes system-wide | Redis write latency / queue depth alert | Hybrid push/pull threshold; partial fan-out + backfill; rate-limited fan-out workers |
| Fan-out worker crash mid-batch | Some followers never receive the post in their cache | Kafka consumer lag; offset not committed | At-least-once via Kafka; idempotent fan-out (dedupe by post_id in target sorted set) |
| Feed Store node/shard failure | Affected users see empty or stale feed | Redis cluster health checks | Replica promotion; on total loss, fall back to pull-model reconstruction (Section 8) |
| Ranking model server slow/down | Feed requests time out or queue up | Ranking service p99 latency alert | Circuit breaker → fallback to heuristic (recency + affinity) ranker |
| Feature store staleness | Ranking quality silently degrades | Feature freshness / null-rate monitoring | Default feature values on miss; alert if miss rate crosses threshold |
| Read stampede on a single viral post | Post Store hot-partition, elevated read latency | Per-key read rate monitoring | Request coalescing; CDN/edge cache for hydrated post content |
| Feed drift during pagination | Duplicate or missing posts across pages | User reports / QA | Cursor-based pagination keyed on `(rank_score, post_id)`, not offset |
| Stale privacy/block-list cache | Blocked user's content briefly visible | Privacy-violation reports; audit sampling | Synchronous consistent read for visibility checks at hydration time (Section 10) |
| Post-events Kafka lag | Fan-out and freshness fall behind post creation | Consumer lag monitoring | Scale fan-out worker fleet; add partitions; apply backpressure upstream if needed |
| Ad billing double-count | Advertiser overcharged | Reconciliation job discrepancy | Ledger is a separate CP system with idempotent impression IDs, decoupled from serving |
| Social graph read overload during celebrity fan-out | Graph service latency spikes, stalls fan-out | Graph service QPS/latency alert | Stream follower list in pages rather than materializing in memory (Section 6) |

---

## 16. Scalability & Sharding

| Component | Sharding / Scaling Strategy |
|---|---|
| Post Store | Partition by `author_id` (co-locates an author's posts for pull-model reads and profile views) |
| Feed Store (Redis) | Partition by `viewer_id`, Redis Cluster consistent hashing |
| Social Graph | Partition by `user_id` for both forward and inverse indexes; heavy read-replication |
| Fan-out workers | Kafka `post-events` partitioned by `author_id`; workers scale horizontally per partition; a single celebrity post's fan-out is further parallelized internally across the follower list |
| Ranking / feature serving | Stateless inference nodes behind a load balancer, autoscaled on request volume/CPU-GPU utilization |
| Feature Store (online) | Partitioned KV store (e.g., by `user_id`), written by the streaming feature pipeline |
| Object storage / CDN | Managed, scales automatically; regional edge caching for media |

---

## 17. Feed Freshness & Real-Time Signal Pipeline

### The Freshness Spectrum

| Path | Typical delay | Why |
|---|---|---|
| Push fan-out (regular authors) | Seconds | Direct write into follower feed caches on post creation |
| Pull fan-out (celebrities) | Effectively real-time at read | Computed live at request time, never stale by design |
| "New posts available" banner | Lightweight polling / long-poll signal | Avoids pushing full content over a persistent connection to every follower; client re-fetches on demand |
| Real-time ranking signals (likes/comments in the last few minutes) | Seconds | Kafka → Flink stream processing → online feature store |
| Batch ranking signals (long-run affinity) | Hours–1 day | Nightly Spark/Flink aggregation job |

### Why Not Push Full Content Over WebSockets to Everyone

Continuously streaming full post content to every follower's open client doesn't scale the same way a lightweight "your feed has updates" signal does — the former is proportional to (followers × content size), the latter is a constant-size ping that lets the client decide whether/when to pay the cost of a real fetch. This mirrors the crawler guide's Kafka backpressure principle: decouple "something changed" from "here is all the data," and let the consumer pull the expensive part on its own schedule.

---

## 18. Senior vs Staff Answer Differentiators

### Senior-level expectations

- Identify push, pull, and hybrid fan-out and pick hybrid
- Know Redis is used for a precomputed feed cache
- Know ranking involves ML, at a high level
- Mention sharding the social graph and feed store
- Handle obvious failure modes (node crash, cache miss)
- Provide basic capacity math

### Staff-level expectations (additional)

| Topic | What Staff-level adds |
|---|---|
| Fan-out | Names the exact celebrity threshold mechanism and explains why it bounds *both* sides of the hybrid simultaneously |
| Celebrity problem | Proactively raised, framed as the same failure-mode *class* as hot-key/hot-partition problems elsewhere, with a menu of graduated mitigations |
| Ranking | Two-stage pipeline (candidate generation vs. scoring) with explicit latency budgets; online/offline feature store split; explore/exploit to prevent filter-bubble feedback loops |
| Pagination | Cursor-based on `(rank_score, post_id)`, with an explicit explanation of the feed-drift failure it prevents |
| Privacy | Identifies the block-list/visibility check as the one component that should trade latency for consistency, against the grain of the rest of the AP-leaning system |
| Ads | Explicitly separates serving (AP) from billing (CP) as two different systems, not one |
| Failure handling | Graceful-degradation ladder for ranking (fallback to heuristic ranker), not just "the service might fail" |
| CAP/PACELC | Explicit per-component table, including the deliberate exception, not a single system-wide label |
| Freshness | Distinguishes push-path freshness from pull-path freshness from soft real-time signaling, and explains why full-content push doesn't scale |
| Failure modes | Proactively enumerates fan-out storms, read stampedes, feed drift, privacy staleness — before being asked |

### The single most important Staff differentiator

Recognizing that **the celebrity problem and the privacy correctness trap are the same kind of thing this library has flagged in every other domain** — an asymmetric, disproportionate-impact edge case that a uniform design misses. A Staff engineer doesn't just solve the news-feed-specific instance of that pattern; they name the pattern.

---

## 19. Interview Time Allocation

For a 45-minute system design session:

| Phase | Time | Focus |
|---|---|---|
| Requirements & scope | 5 min | Confirm functional + non-functional; 500M DAU target |
| Capacity estimation | 5 min | Posts/sec, reads/sec, read:write ratio, feed cache footprint |
| High-level architecture | 5 min | Draw write path + read path; name Kafka as the decoupling layer |
| Fan-out design (deep dive) | 12 min | Push vs pull vs hybrid; threshold mechanism; celebrity problem |
| Ranking (deep dive) | 10 min | Candidate generation vs scoring; feature store split; explore/exploit |
| Feed aggregation & privacy | 5 min | Cursor pagination; block-list consistency exception |
| Failure modes | 5 min | Fan-out storms, cache loss, ranking fallback, feed drift |
| Ads / storage (if time) | 3 min | Serving vs billing split; media/CDN briefly |

**What to cut if short on time:** ad system detail, media/CDN depth (defer to the YouTube guide). **Never cut:** the fan-out tradeoff, the celebrity problem, or the privacy consistency exception — these are the three questions most likely to anchor the interviewer's deep-dive.

---

## 20. Cross-References to Other Guides in This Library

| This guide relies on / pairs with | For |
|---|---|
| **Distributed Unique ID Generation** | `post_id` generation (Snowflake-style, time-ordered, no coordination) |
| **Apache Kafka** | `post-events` topic design, WAL-as-durability-primitive reasoning, backpressure and replay |
| **Sharding Strategies** | Partitioning schemes applied here to Post Store (by author_id), Feed Store (by viewer_id), Social Graph (by user_id) |
| **CAP / PACELC Theorem** | Foundational theory underlying Section 14's per-component table |
| **Circuit Breaker Pattern** | The ranking-service fallback path (Section 9) is a textbook circuit breaker application |
| **Notification System** | "New post" push notifications can be triggered off the same `post-events` stream consumed by fan-out |
| **YouTube Video Streaming** | Transcoding ladder, adaptive bitrate, and CDN specifics for video posts (out of scope here) |
| **Twitter/Feed Design** (companion guide) | Deeper dive on retweet/reshare-specific fan-out mechanics and a more chronology-leaning timeline model; this guide's ranking and celebrity sections generalize to that design |
| **API Gateway** | Sits in front of both the write path (post creation) and read path (feed fetch) shown in Section 4 |

**Suggested reconciliation pass:** the celebrity fan-out threshold here and the "hot domain" sharding logic in the Web Crawler guide's frontier are structurally the same mitigation pattern (bound the outlier, don't redesign for the average case) — worth a shared note if you build a cross-cutting "hot-key patterns" companion doc later.

---

## 21. Quick-Reference Cheatsheet

```
KEY DECISIONS
─────────────
Fan-out:         Hybrid — push for <10K followers, pull for celebrities
Feed cache:      Redis sorted set per viewer, capped ~800 entries
Ranking:         Two-stage — candidate generation (recall) → ML scoring (precision)
Pagination:      Cursor on (rank_score, post_id), never offset-based
Privacy:         Synchronous, consistent block-list/visibility check at hydration

KEY NUMBERS (illustrative)
───────────────────────────
500M DAU, 100M posts/day        →  1,157 posts/sec
10B feed reads/day               →  115,740 reads/sec (347K peak)
Read:Write ratio                 ≈  100:1
Naive fan-out (no hybrid)        ≈  231K writes/sec
Feed cache footprint             ≈  6.4 TB across Redis cluster
Celebrity post fan-out           up to 50M+ writes for ONE event

KEY SYSTEMS
───────────
Write path:      Post Creation → Post Store (Cassandra, by author_id)
                  → Kafka (post-events) → Fan-out Service
Read path:        Feed Aggregation → Feed Store (Redis) + Pull Fan-out
                  → Ranking Service → Hydration → Client
Feature store:    Offline (batch affinity) + Online (streaming, Flink)
Media:            Object store + CDN, referenced not embedded
Ads:              Serving (AP) decoupled from Billing ledger (CP)

CAP / PACELC DECISIONS
───────────────────────
AP / EL:  Feed Store, Post Store, Social Graph, Feature Store,
          Ranking serving, Ad serving, Kafka fan-out queue
CP / EC:  Privacy/block-list check, Ad billing ledger,
          fan-out worker coordination (if using ZK/etcd)

THE CELEBRITY PROBLEM — MITIGATION LADDER
───────────────────────────────────────────
1. Hybrid push/pull threshold (primary fix)
2. Partial fan-out to active users + async backfill
3. Priority fan-out queue (online users first)
4. Parallel fan-out workers per single post
5. Read-side request coalescing for viral post hydration
6. Rate-limit fan-out throughput into Feed Store

STAFF-LEVEL SIGNALS TO HIT
───────────────────────────
✓ Hybrid fan-out with explicit, justified threshold
✓ Celebrity problem raised proactively, framed as a general hot-key pattern
✓ Two-stage ranking pipeline with explicit latency budget
✓ Online/offline feature store split
✓ Cursor-based pagination and why offset-based breaks
✓ Privacy check named as the deliberate CP exception in an AP system
✓ Ad serving vs. ad billing explicitly separated
✓ Graceful-degradation ladder for ranking service failure
✓ Explicit CAP/PACELC decision per component, not system-wide
✓ Proactive failure mode enumeration (don't wait to be asked)
```

---

*Guide compiled for Senior & Staff-level system design interview preparation.*
*Topics: News Feed, Fan-out Strategies, Ranking Systems, Feature Stores, CAP/PACELC Theorem, Celebrity Problem.*
*Companion guide: Twitter/Feed Design. Related: Web Crawler (hot-key/spider-trap pattern parallel), Apache Kafka, Notification System, Distributed Unique ID Generation, Sharding Strategies, Circuit Breaker Pattern, YouTube Video Streaming.*
