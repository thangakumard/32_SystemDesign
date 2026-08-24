# Design Twitter — System Design Interview Guide
### Senior & Staff Engineer Level (v2 — incorporates request-lifecycle walkthrough, security, observability & testing)

---

## Table of Contents

1. [Problem Statement & Scope](#1-problem-statement--scope)
2. [Requirements](#2-requirements)
3. [Capacity Estimation](#3-capacity-estimation)
4. [High-Level Architecture](#4-high-level-architecture)
5. [Component 1: Clients, Load Balancer & API Gateway](#5-component-1-clients-load-balancer--api-gateway)
6. [Component 2: Auth Service](#6-component-2-auth-service)
7. [Component 3: Tweet Ingestion Service](#7-component-3-tweet-ingestion-service)
8. [Component 4: Reply Service](#8-component-4-reply-service)
9. [Component 5: Follow Graph / Profile Service](#9-component-5-follow-graph--profile-service)
10. [Component 6: Timeline / Feed Generation (Fanout)](#10-component-6-timeline--feed-generation-fanout)
11. [Component 7: Timeline Read Path](#11-component-7-timeline-read-path)
12. [Component 8: Ranking Service](#12-component-8-ranking-service)
13. [Component 9: Media Storage & CDN](#13-component-9-media-storage--cdn)
14. [Component 10: Search Service](#14-component-10-search-service)
15. [Component 11: Trending Topics](#15-component-11-trending-topics)
16. [Component 12: Notification Service](#16-component-12-notification-service)
17. [Component 13: Counters (Likes / Retweets / Views)](#17-component-13-counters-likes--retweets--views)
18. [CAP Theorem Positioning](#18-cap-theorem-positioning)
19. [Failure Modes & Mitigations](#19-failure-modes--mitigations)
20. [Scalability & Sharding](#20-scalability--sharding)
21. [The Celebrity Problem (Hybrid Fanout Deep Dive)](#21-the-celebrity-problem-hybrid-fanout-deep-dive)
22. [Security](#22-security)
23. [Observability: Monitoring, Logging & Alerting](#23-observability-monitoring-logging--alerting)
24. [Testing & CI/CD](#24-testing--cicd)
25. [Senior vs Staff Answer Differentiators](#25-senior-vs-staff-answer-differentiators)
26. [Interview Time Allocation](#26-interview-time-allocation)
27. [Quick-Reference Cheatsheet](#27-quick-reference-cheatsheet)

---

## 1. Problem Statement & Scope

Design **Twitter** (a microblogging / social feed platform): users create accounts, post short text/media updates ("tweets"), follow other users, and read a **timeline** aggregating tweets from accounts they follow. The system must also support likes, retweets, replies, search, trending topics, and notifications.

### What the interviewer is really testing

- Can you decompose a large product into a clean **microservice boundary** and justify each cut?
- Can you handle **massive fan-out / read-heavy asymmetry** (read:write ratio ~1000:1)?
- Do you understand the **push vs pull tradeoff** for feed generation, and the **celebrity/hot-key problem**?
- Can you design for **eventual consistency** where it's acceptable, and be precise about where it isn't?
- Do you round out the design with **security, observability, and testing** instead of treating the interview as "done" once the data flows?

### Explicit non-goals (state these to control scope)

- Direct messages (separate messaging system — different consistency/delivery requirements)
- Ads serving / ad auction (separate system)
- Payments / monetization
- Spaces (live audio) — real-time streaming is a different problem class
- Recommendation-based ("For You") timeline ranking beyond a lightweight scoring pass — full ML recommendation is its own interview

Twitter's real system took **17 years** to build; you have 45 minutes. Naming what you're deliberately excluding, and getting a quick nod from the interviewer, is itself a Staff-level signal — it shows you can bound a problem rather than trying to design everything. **Spend no more than ~5 minutes on requirements** — it's necessary groundwork, not the interesting part of the interview, and over-investing here is one of the most common ways candidates run out of time later.

---

## 2. Requirements

### Functional Requirements

| Requirement | Notes |
|---|---|
| Create an account and log in | Auth is a first-class concern, not an afterthought |
| Post, edit, and delete a tweet (text ≤280 chars, media, poll) | Core write path |
| Follow / unfollow a user | Graph mutation |
| View home timeline (tweets from followed accounts) | Core read path — the hard problem |
| View user timeline (tweets by one user) | Simpler — single-partition read |
| Like, retweet, reply, quote-tweet | Engagement actions; counters |
| Search tweets by keyword/hashtag | Full-text search |
| See trending topics | Real-time aggregation |
| Receive notifications (like, follow, reply, mention) | Near-real-time push |

### Non-Functional Requirements

| Requirement | Target |
|---|---|
| Scale | Hundreds of millions of DAU (~300M+), ~500M tweets/day |
| Availability | 99.99% uptime — a down timeline is a worse outage than a stale one |
| Read:write ratio | ~1000:1 (timeline reads dwarf tweet writes) |
| Latency | Timeline reads and tweet loads should feel instantaneous — p99 < 200ms on read, p99 < 500ms on write |
| Security & privacy | User data protected at rest and in transit |
| Consistency | Eventual consistency acceptable almost everywhere; strong consistency needed only for a few specific flows (see CAP section) |

---

## 3. Capacity Estimation

```
Users: 300M DAU

Writes (tweets):
  500M tweets/day → 500,000,000 / 86,400 ≈ 5,800 tweets/sec avg
  Peak (3x avg, e.g. major live event) ≈ 17,400 tweets/sec

Reads (timeline loads):
  Assume each DAU refreshes timeline ~10x/day
  300M × 10 = 3B timeline reads/day
  3,000,000,000 / 86,400 ≈ 34,700 reads/sec avg
  Peak (3x) ≈ 104,000 reads/sec

  Read:Write ratio ≈ 34,700 : 5,800 ≈ 6:1 at the request level
  BUT each timeline read touches ~200-800 source tweets via fanout,
  so the *effective* read amplification is far higher — this is
  why fanout strategy dominates the design.

Storage (tweet document, avg 300 bytes text + metadata):
  500M/day × 300 bytes ≈ 150 GB/day text
  5 years retention ≈ 270 TB (metadata only, excludes media)

Media (assume 20% of tweets have media, avg 200KB):
  500M × 0.2 × 200KB = 20 TB/day  →  object store + CDN, not DB

Follow graph edges:
  Avg 200 follows/user × 300M users = 60B edges
  At ~20 bytes/edge ≈ 1.2 TB — fits in a distributed graph/KV store

Fanout write amplification (push model):
  Avg followers per tweet author: ~700 (heavily skewed — median is
  low tens, but a small population of celebrities has 100M+)
  5,800 tweets/sec × 700 avg fanout ≈ 4M timeline-cache writes/sec
  at peak — this number is why the celebrity problem needs a
  separate strategy (see Section 21).

Cache sizing (home timeline cache, last ~800 tweet IDs per user):
  300M users × 800 tweet IDs × 8 bytes ≈ 1.9 TB → Redis cluster
```

**Key insight:** Unlike a storage-bound system, Twitter is **fanout-bound**. The dominant cost is not storing tweets — it's the write amplification of pushing each tweet into potentially millions of follower timelines. This single fact drives almost every architectural decision in the feed system.

---

## 4. High-Level Architecture

```
                     ┌──────────────┐
                     │   Clients     │
                     │ (web/iOS/    │
                     │  Android)    │
                     └──────┬───────┘
                            │
                     ┌──────▼───────┐
                     │ Load Balancer │
                     │ (L7, round-   │
                     │  robin)       │
                     └──────┬───────┘
                            │
                     ┌──────▼───────┐
                     │ API Gateway   │
                     │ (routes to    │
                     │  microservice)│
                     └──────┬────────┘
                            │
    ┌─────────┬─────────────┼──────────────┬─────────────┐
    │         │             │              │             │
┌───▼───┐ ┌───▼────┐  ┌─────▼─────┐  ┌─────▼─────┐ ┌─────▼─────┐
│ Auth  │ │ Tweet   │  │ Reply     │  │ Timeline  │ │ Profile   │
│Service│ │ CRUD    │  │ Service   │  │ Service   │ │ Service   │
└───┬───┘ └───┬─────┘  └─────┬─────┘  └─────┬─────┘ └─────┬─────┘
    │         │              │              │             │
    ▼         ▼              ▼              ▼             ▼
┌───────┐ ┌────────┐   ┌──────────┐  ┌───────────┐  ┌───────────┐
│ User  │ │ Tweet   │   │ Reply DB  │  │ Timeline  │  │ SQL (user)│
│ Auth  │ │ Store   │   │ (indexed  │  │ Cache     │  │ + Graph DB│
│ Store │ │(NoSQL/  │   │  by       │  │ (Redis)   │  │ (follows) │
│       │ │wide-col)│   │  tweet_id)│  │           │  │           │
└───────┘ └───┬─────┘   └──────────┘  └───────────┘  └───────────┘
              │
              │ (Kafka / message queue: new-tweets)
              ▼
    ┌─────────────────────────────────────────┐
    │              Fanout Service               │
    │  hybrid push/pull to follower timelines   │
    └───┬─────────────┬─────────────┬───────────┘
        │             │             │
        ▼             ▼             ▼
  ┌──────────┐  ┌───────────┐ ┌─────────────┐
  │Search Idx │  │ Trending   │ │Notification │
  │(Elastic-  │  │ Aggregator │ │ Service     │
  │search,    │  │            │ │             │
  │via CDC)   │  │            │ │             │
  └──────────┘  └───────────┘ └─────────────┘

  CDN sits in front of the client for static assets, media, and
  frequently-accessed tweets (edge closer to the user).

  ELK stack (logs) + Prometheus/Grafana (metrics) + Alertmanager/
  PagerDuty (alerts) attach to every service — omitted from the
  diagram above for clarity, but present on every component.
```

Message queue / **Kafka** is the backbone connecting the write path to every downstream consumer (fanout, search indexing via CDC, trending aggregation, notifications). This decouples ingestion from fanout latency and lets each consumer scale and fail independently.

---

## 5. Component 1: Clients, Load Balancer & API Gateway

These three components appear in roughly 90% of system design interviews — when in doubt, they're a safe, expected starting point that lets you move left-to-right across the request lifecycle.

### Clients

Web app and native mobile (iOS/Android). Client-side responsibilities worth naming explicitly: basic input validation/sanitization (defense in depth — never trust the client alone, see Security section) and rendering the CDN-served static assets.

### Load Balancer

Distributes incoming traffic across backend server fleets — key to scalability in nearly every design. Two dimensions worth discussing explicitly in an interview:

**Routing algorithm:**

| Algorithm | How it works | When to use |
|---|---|---|
| Round robin | Rotates requests evenly across servers | Simple, fair, no persistent-connection requirement, stateless backends — **our choice for Twitter** |
| Least connections | Routes to the server with the fewest active connections | Backend request durations vary widely |
| IP hash | Same client IP always routes to the same server | Needed only if you require session affinity (we don't — our services are stateless) |

**OSI layer:**

| Layer | Operates on | Tradeoff |
|---|---|---|
| Layer 4 (transport, TCP) | IP + port only | Faster, but can't inspect request content |
| Layer 7 (application, HTTP/HTTPS) | Full request — URL, headers, body | Slightly more overhead, but enables content-based routing for canary/feature rollouts and richer traffic management as the system scales — **our choice for Twitter** |

### API Gateway

Because the system is decomposed into microservices (each independently responsible for one piece of functionality — Tweet CRUD, Replies, Search, Timeline, Profile, Auth), a single entry point is needed to route each incoming request to the correct backend service. The API Gateway also centralizes cross-cutting concerns that would otherwise be duplicated per-service: IP-based rate limiting (DDoS protection — see Security), authentication token validation (delegating to the Auth Service), and request logging.

**Why microservices instead of a monolith here?** Twitter's core operations — writing a tweet, fanning out a timeline, running a search query, sending a notification — have wildly different scaling profiles (fanout is write-amplification-bound; search is index-bound; timeline reads are cache-bound). Splitting them lets each scale, deploy, and fail independently, at the cost of added operational complexity (service discovery, distributed tracing, network hops). This tradeoff is worth stating explicitly rather than assuming microservices are free.

---

## 6. Component 2: Auth Service

A dedicated service, separate from the Profile Service, handling authentication (is this really you?) and authorization (are you allowed to do this?).

### Why separate it from Profile Service?

- **Security isolation** — a narrower, more heavily scrutinized surface area is easier to audit and harden than logic tangled into general user-profile CRUD
- **Focused maintenance** — auth standards (OAuth flows, token rotation, MFA) evolve independently of profile features
- **Reusability** — every other service (Tweet CRUD, Reply, Profile) needs to validate a caller's identity; centralizing this avoids duplicating auth logic across services and gives a single place to integrate third-party identity providers later

### Responsibilities

```
- Account creation / login (credential verification)
- Session/token issuance (JWT or opaque token + session store)
- Token validation on every authenticated request (typically
  short-circuited at the API Gateway before forwarding downstream)
- Password hashing (bcrypt/argon2) and credential storage,
  isolated from general user profile data
```

---

## 7. Component 3: Tweet Ingestion Service

### Responsibilities

1. Validate tweet (length, rate limit, spam/abuse checks)
2. Assign a globally unique, roughly time-sortable `tweet_id`
3. Upload any attached media to object storage first, store a reference
4. Write the tweet document to the durable Tweet Store
5. Publish to the message queue (`new-tweets` topic) for downstream fanout, search indexing, and trending aggregation

### Tweet ID Generation — Snowflake IDs

A **Staff-level must-know**. Never use auto-increment (single point of contention) or pure UUID (not sortable, poor index locality).

```
64-bit Snowflake ID layout:
┌─┬───────────────────────────────┬──────────────┬───────────┐
│0│  41 bits: timestamp (ms since │ 10 bits:      │ 12 bits:  │
│ │  custom epoch, ~69 yr range)  │ machine/shard │ sequence  │
│ │                                │ ID            │ (per-ms)  │
└─┴───────────────────────────────┴──────────────┴───────────┘

Properties:
  • Roughly sortable by creation time (useful for range scans)
  • No coordination needed between generator nodes (each has own
    machine ID)
  • 4096 IDs/ms per node before sequence overflow — plenty
```

### Tweet Store — Two Valid Approaches

State the choice explicitly and justify it — interviewers care more about your reasoning than which specific product you pick.

**Option A — Document store (MongoDB or similar NoSQL document DB):**

```json
{
  "tweet_id": "1823...",
  "author_id": "u_9284",
  "content": "text of the tweet",
  "created_at": "2026-08-04T10:30:00Z",
  "hashtags": ["systemdesign"],
  "mentions": ["u_1123"],
  "location": {"lat": 47.6, "lng": -122.2},
  "media": ["s3://media/abc123.jpg"],
  "like_count": 0,
  "retweet_count": 0
}
```

A tweet has no need for complex joins — reading a tweet means fetching one self-contained document. Document stores are a strong fit for high-throughput, low-latency single-document reads/writes with a naturally nested schema (hashtags, mentions, media all in one object).

**Option B — Wide-column store (Cassandra):**

```sql
CREATE TABLE tweets_by_user (
  user_id       BIGINT,
  tweet_id      BIGINT,   -- snowflake, sortable
  content       TEXT,
  media_urls    LIST<TEXT>,
  reply_to_id   BIGINT,
  created_at    TIMESTAMP,
  PRIMARY KEY (user_id, tweet_id)
) WITH CLUSTERING ORDER BY (tweet_id DESC);
```

Partitioned by `user_id` → a user's own tweets are collocated → efficient "user profile timeline" range scans (`WHERE user_id = ? ORDER BY tweet_id DESC LIMIT 20`) without an extra index, which a pure document store doesn't give you for free.

**Which to pick:** if user-profile timeline range-scans and multi-region tunable consistency are priorities, lean Cassandra. If schema flexibility (evolving tweet fields, e.g. polls, quote-tweet metadata) and operational simplicity are priorities, lean document store. Either way, media itself is **never** stored in the primary datastore — see Section 13.

### Rate Limiting on the Write Path

A token-bucket rate limiter per user sits on the tweet-creation path (distinct from the API Gateway's IP-based DDoS rate limiting — this one is account-based and specifically caps tweet/reply creation frequency) to prevent bot-driven flooding from degrading write throughput for everyone else.

### Write Path

```
1. Client → Load Balancer → API Gateway → Tweet Ingestion Service
2. Validate + per-user rate limit (token bucket)
3. Generate tweet_id (Snowflake)
4. Upload media to S3 (if present) → get back a reference URL
5. Write tweet document (with media reference) to Tweet Store —
   durable source of truth
6. Publish {tweet_id, author_id, created_at} to message queue:
   new-tweets
7. Return 201 to client immediately (don't block on fanout)
```

Fanout, search indexing, and trending aggregation all happen **asynchronously** after the durable write — this decouples write latency from follower count and downstream consumer load.

---

## 8. Component 4: Reply Service

Replies get their **own CRUD service and their own datastore**, separate from the Tweet Store, rather than being nested inside the tweet document.

### Why split replies out as a separate service?

| Reason | Detail |
|---|---|
| Scalability | Popular tweets can gather thousands of replies. Embedding them would make the parent tweet document unbounded and unwieldy. A separate service also lets replies scale independently — critical during a viral moment where reply volume spikes far faster than new-tweet volume |
| Read performance | Fetching a tweet should not require loading all of its replies. Keeping them separate keeps the base tweet read fast; replies are fetched (and paginated) on demand — this mirrors the real product UX, where you see the first N replies and only fetch more on scroll |

### Data Model

```json
{
  "reply_id": "r_88213",
  "tweet_id": "1823...",     // indexed — this is the critical field
  "author_id": "u_5521",
  "content": "text of the reply",
  "created_at": "2026-08-04T10:32:00Z"
}
```

The reply store is **indexed by `tweet_id`** so that fetching a tweet's replies is a fast indexed lookup, not a scan.

### Read Path Nuance

Even though replies live in a separate store, they're **bundled with the tweet at read time** — when a client requests a tweet, the Timeline/Tweet read path also fetches (and caches) the first page of replies from the Reply Service so the client doesn't need a second round trip for the common case of "show me this tweet with its top replies."

### Write Path

Same shape as tweet ingestion: per-user rate limiter → write to Reply Store → publish an event (for reply-count updates and reply notifications) → return immediately.

---

## 9. Component 5: Follow Graph / Profile Service

### Data Storage — Two Complementary Stores

**User profile data → SQL (relational):**

```sql
CREATE TABLE users (
  user_id     BIGINT PRIMARY KEY,
  username    VARCHAR(50) UNIQUE,
  email       VARCHAR(255) UNIQUE,
  bio         TEXT,
  created_at  TIMESTAMP
);
```

SQL is the right choice here specifically because: profile attributes are structured and benefit from schema enforcement; ACID transactions matter for things like username uniqueness and email changes; and complex joins/aggregations are genuinely useful for user analytics — none of which apply to the tweet-writing hot path, which is why tweets themselves don't live here.

**Follow relationships → Graph database:**

Graph databases are purpose-built for network/relationship data — mapping "who follows whom" and traversing social ties is exactly what they're optimized for. As the product grows, this same graph becomes the substrate for recommendation features (people-you-may-know, suggested follows via multi-hop traversal).

```
(User A) -[:FOLLOWS]-> (User B)
(User A) -[:FOLLOWS]-> (User C)
```

### Alternative: Sharded Key-Value Store

A graph database is the natural first answer, but it's worth naming the tradeoff explicitly: Twitter's actual query patterns are almost entirely **single-hop** ("give me X's followers", "does X follow Y"), which a much simpler sharded KV store serves at lower operational cost:

```sql
CREATE TABLE following (      -- "who does user X follow?"
  user_id BIGINT, followee_id BIGINT, created_at TIMESTAMP,
  PRIMARY KEY (user_id, followee_id)
);
CREATE TABLE followers (      -- "who follows user X?"
  user_id BIGINT, follower_id BIGINT, created_at TIMESTAMP,
  PRIMARY KEY (user_id, follower_id)
);
```

**When to justify each:** if the interviewer wants recommendation-adjacent multi-hop traversal as part of scope, lead with the graph DB. If the interview stays tightly scoped to "who follows whom" lookups for fanout purposes, the sharded KV store is simpler to operate at scale and is the better-justified choice. Either way, `hash(user_id) % N` sharding keeps a user's full follower/following list on one shard (or a bounded set of sub-shards for a celebrity — see Section 21).

### Consistency Note

Follow/unfollow is **read-your-own-writes** critical (a user must see their own follow take effect immediately) but **AP** with respect to propagation into the fanout system — a few seconds of delay before a new follow's timeline backfills is acceptable.

---

## 10. Component 6: Timeline / Feed Generation (Fanout)

This is **the heart of Twitter's design** — go deep here regardless of which other sections you have to cut short.

### The Core Tradeoff: Fanout-on-Read vs Fanout-on-Write

```
FANOUT-ON-READ (pull, naive baseline)
─────────────────────────────
On tweet publish:  just write the tweet — O(1), cheap
On timeline read:
  1. Query the list of accounts the user follows
  2. Fetch tweets from every one of those accounts
  3. Sort by time, return

This is the simplest possible design and is exactly what most
candidates propose first. It clearly fails the low-latency
non-functional requirement: for a user following hundreds or
thousands of accounts, every timeline load becomes an expensive
fan-in read across many partitions, computed synchronously on
the request path.


FANOUT-ON-WRITE (push)
─────────────────────────────
On tweet publish:
  1. Place the new tweet onto a message queue immediately
     (buffers bursts — many tweets are created at the same
     instant, and we don't want to overwhelm downstream work)
  2. Fanout workers continuously pull tweets off the queue
  3. For each tweet, fetch the author's follower list
  4. For each follower, prepend the tweet to that follower's
     timeline cache — a fast key-value store holding each
     user's most recent feed
On timeline read:
  Just read the precomputed cache — O(1), fast

This is more write-intensive but makes timeline reads lightning
fast, because the data is already assembled and waiting. It's a
deliberate trade-off that prioritizes read speed — which is
exactly what our non-functional requirements call for, given a
read:write ratio around 1000:1.


HYBRID MODEL (needed for mega-accounts) — Staff-level answer
─────────────────────────────
Fanout-on-write works well for the average user, but for
accounts with millions of followers (a celebrity, a major news
account), the sheer volume of per-follower cache updates for a
single tweet can overwhelm the fanout pipeline. For these
accounts, fall back to fanout-on-read: don't push proactively;
instead, when a follower of that account loads their timeline,
fetch the mega-account's latest tweets live and merge them in.
See Section 21 for the full mechanics.
```

### Fanout Service Pipeline

```
Message queue: new-tweets
      │
      ▼
┌─────────────────────────────────────┐
│  Fanout Worker (consumer group)      │
│  1. Read {tweet_id, author_id}       │
│  2. Classify author:                 │
│       follower_count < threshold?    │
│         → PUSH path                  │
│       follower_count ≥ threshold?    │
│         → SKIP push (handled at      │
│           read time instead)         │
│  3. If PUSH:                         │
│       fetch follower list (paginated)│
│       for each follower:             │
│         prepend tweet_id to their    │
│         timeline cache               │
│       trim cache to last ~800 IDs    │
└───────────────────────────────────────┘
```

### Timeline Cache Structure (Redis)

```
Key:   timeline:{user_id}
Value: sorted list of tweet_ids, most recent first, capped at ~800

LPUSH timeline:{user_id} {tweet_id}
LTRIM timeline:{user_id} 0 799   -- bound memory per user
```

Timeline cache stores IDs only, not full tweet content — content is hydrated from the Tweet Store / a tweet-content cache at read time. This keeps the hot cache small and lets popular tweet content be reused across every follower's timeline instead of duplicated per-follower.

### Staff-Level Gotchas

**Dormant user waste:** Pure push wastes compute fanning out to users who haven't opened the app in months. Fix: track last-active timestamp; skip/defer fanout writes for users inactive beyond N days, and rebuild their timeline via pull on their next login instead.

**Fanout worker backpressure:** A viral tweet from a mid-tier account can still create a fanout spike. Fix: fanout workers are a consumer group that scales horizontally; apply consumer-lag-triggered backpressure.

**Cache eviction storms:** If the timeline cache evicts many users' entries simultaneously under memory pressure, their next read becomes an expensive full pull-rebuild — a cache stampede. Fix: staggered TTLs, and a rebuild queue with per-user locking so concurrent requests for the same evicted user don't all trigger redundant rebuilds.

---

## 11. Component 7: Timeline Read Path

```
1. Client requests home timeline for user_id
2. Timeline Service:
     a. Fetch precomputed tweet_id list from the timeline cache
     b. Fetch the list of mega-accounts this user follows
     c. Pull recent tweets from each of those mega-accounts
        (bounded — most users follow only a handful, so this
        merge is small, not a full fan-in)
     d. Merge push results + pull results by timestamp
     e. Pass merged candidate list to Ranking Service (Section 12)
     f. Hydrate final tweet IDs → full content (batch multi-get
        from a tweet content cache, fall back to the Tweet Store
        on cache miss)
3. Return ranked, hydrated timeline to client
```

### Caching on the Read Path

Two distinct caching layers, each solving a different problem:

- **Timeline cache** (Redis/Memcached) — "which tweet IDs belong on this user's feed," precomputed by fanout
- **Tweet content cache** — "what's actually inside a given tweet," populated on demand and reused across every follower who has that tweet_id in their timeline

Additionally, a **CDN** sits in front of the client for static assets, media, and — importantly — frequently-accessed tweets themselves. Caching popular tweet content at edge locations closer to users' geography further reduces latency for a global user base, on top of the origin-side caches above.

### Pagination

Cursor-based, not offset-based: `next_cursor = last_seen_tweet_id`. Offset pagination degrades badly on a live, constantly-growing list; cursor-based pagination is O(1) regardless of depth.

---

## 12. Component 8: Ranking Service

### Senior-level answer
Reverse-chronological order, made up entirely of tweets from people you follow (explicitly out of scope: a "For You"-style recommendation engine, which is a separate interview problem). Simple, defensible.

### Staff-level answer
A lightweight **candidate generation → feature scoring → re-ranking** pass, decoupled from the fanout system:

```
Candidates: merged push + pull results, typically ~500-1500
            tweet candidates per timeline load

Feature scoring (low-latency model — e.g. gradient-boosted
trees or a small learned ranker served via an inference
service):
  • Author affinity (how often this user engages with this author)
  • Recency decay
  • Predicted engagement probability
  • Content/author diversity (avoid one account dominating the feed)

Output: re-ordered list
```

**Key architectural point:** ranking is a separate, swappable service consuming the candidate list — lets you A/B test ranking without touching fanout or caching, and lets ranking fail open (fall back to reverse-chronological) without taking down the timeline.

---

## 13. Component 9: Media Storage & CDN

```
Upload path:
  Client → Tweet Ingestion Service → Object Store (S3 or
  equivalent Blob storage) → async transcoding (multiple
  resolutions/formats) → CDN origin pull on first request,
  cached at edge thereafter
```

Media is stored as a **reference** inside the tweet document/row — never embedded directly. Object stores like S3 are purpose-built to handle vast amounts of unstructured binary data and keep retrieval fast and seamless at scale — whenever a design involves media, storing it in Blob storage rather than the primary datastore is almost always the right call.

CDN caching handles read amplification for both media and, as noted in Section 11, popular tweet content: fetched from origin once, served from edge millions of times.

---

## 14. Component 10: Search Service

```
Message queue / CDC  ──▶  Search Indexer (consumer)  ──▶  Elasticsearch

Reverse indexes on: tweet content, username, hashtag
```

The simplest possible approach — scanning the Tweet Store for matching text on every query — doesn't scale. Elasticsearch, a distributed search engine purpose-built for large datasets with low latency and high reliability, is the standard answer whenever full-text search comes up in an interview.

### Keeping the Index in Sync — Change Data Capture (CDC)

Rather than having the Tweet Ingestion Service do a dual write (one to the Tweet Store, one to Elasticsearch — risky, since a partial failure leaves them inconsistent), use **Change Data Capture**: a process that captures and streams changes from the Tweet Store's write-ahead log / oplog into the message queue, which the Search Indexer consumes to keep Elasticsearch continuously up to date. This avoids the dual-write consistency problem and keeps indexing fully decoupled from — and non-blocking for — the tweet write path.

### Query Time

Relevance ranking blends text match score with recency and engagement velocity — a two-minute-old tweet with 10,000 retweets should outrank one that's two minutes old with zero engagement.

---

## 15. Component 11: Trending Topics

### The Problem

Identify hashtags/phrases with rapidly *increasing* frequency — not just highest absolute count (which would always be generic terms).

### Approach: Sliding Window Count + Rate of Change

```
Stream: new-tweets → Trending Aggregator

Maintain, per hashtag/topic, per short time bucket (e.g. 1 min):
  count[topic][bucket] via a streaming aggregation framework
  (Kafka Streams / Flink)

Trending score ≈ current_window_count vs. baseline (e.g. same
hour on previous days, or a trailing moving average) — a
spike detector, not raw count.

Regional trending: partition aggregation by geo/locale.

Output: top-N per region, refreshed every 1-5 min, served from
a fast KV store (Redis sorted set).
```

**HyperLogLog** is worth mentioning explicitly for estimating unique-user counts per trending topic (distinct participants, not just tweet count) without storing every user ID.

---

## 16. Component 12: Notification Service

```
Message queue: engagement-events (like, reply, retweet, follow, mention)
      │
      ▼
Notification Fanout Worker
      │
      ├──▶ Persist to Notification Store (per-user inbox)
      ├──▶ Push via APNs/FCM if user has push enabled
      └──▶ Batch/aggregate ("X and 40 others liked your tweet")
           to avoid notification spam for viral tweets
```

**Staff-level nuance:** a viral tweet generates a notification fanout problem structurally identical to the timeline fanout problem — a celebrity's tweet liked by 50,000 people in a minute must not send the author 50,000 individual push notifications. Aggregate and rate-limit at the notification layer using the same "detect high-volume source, switch strategy" pattern as Section 21.

---

## 17. Component 13: Counters (Likes / Retweets / Views)

### The Problem

Hot counters on viral tweets receive extreme concurrent write volume — a naive `UPDATE tweets SET like_count = like_count + 1` serializes on a single row/document and becomes a bottleneck.

### Approach

```
1. Write the raw like/retweet event to the message queue
   (durable, ordered per-key by tweet_id)
2. A stream aggregator batches increments per tweet_id over a
   short window and applies them as a single batched update, or
   maintains counts in a fast in-memory store (sharded by
   tweet_id) that's periodically flushed to durable storage
3. Read path serves counts from the fast store; durable store
   is the reconciliation source of truth, eventually consistent
```

Write-coalescing avoids hot-row contention the same way sharding a search index or a fanout queue avoids a single coordinator bottleneck.

---

## 18. CAP Theorem Positioning

| Component | CAP Choice | Reasoning |
|---|---|---|
| Tweet Store | **AP** | Tunable consistency; optimize for high-throughput writes, accept brief staleness on reads |
| Reply Store | **AP** | Same reasoning as Tweet Store; replies are independently scaled and tolerate brief staleness |
| Timeline Cache | **AP** | A timeline that's a few seconds stale is completely acceptable; availability (always show *a* feed) matters far more |
| Follow Graph | **AP** for propagation into fanout, but **read-your-own-writes** required for the follow action itself |
| User profile data (SQL) | **CP-leaning** | Username/email uniqueness and profile integrity benefit from stronger consistency guarantees than the rest of the system |
| Tweet Content Cache | **AP** | Stale like-counts for a few seconds are fine |
| Counters | **AP** | Eventually consistent; exact real-time precision isn't required |
| Search Index | **AP** | New tweets appearing in search a few seconds late is acceptable |
| Auth / session store | **CP** | Must not allow inconsistent auth state (e.g., a revoked token still validating) — prefer denying access during a partition over allowing it |
| Snowflake ID generation | **Coordination-free by design** | Sidesteps the CAP tradeoff entirely via per-node machine IDs instead of a coordinated counter |

**The Staff-level insight for this table:** almost the entire system is deliberately **AP**. Twitter's core value proposition (a feed, not a ledger) tolerates staleness far better than it tolerates unavailability. The CP exceptions (auth, user profile integrity, rate-limiting/abuse counters) should be named explicitly — showing you know *why* they're different, not just applying AP everywhere by default.

---

## 19. Failure Modes & Mitigations

| Failure | Symptom | Detection | Mitigation |
|---|---|---|---|
| Celebrity tweet fanout storm | Fanout workers fall behind; queue lag spikes | Consumer lag on `new-tweets` | Hybrid push/pull (Section 21); skip push for accounts above follower threshold |
| Hot partition (viral tweet's content cache key) | Single cache node saturates; latency spike | Per-key request rate monitoring | Read-through replica fanout for hot keys; request coalescing |
| Timeline cache eviction storm | Many users' caches evicted simultaneously | Cache hit-rate drop | Staggered TTLs; per-user rebuild locks; graceful degrade to pull-only |
| Notification flood on viral tweet | Push provider rate-limits or overwhelms author's device | Push delivery error rate | Aggregate before sending; rate-limit per recipient |
| Counter write contention | Hot-row/document lock contention on like/retweet count | DB write latency spike on specific keys | Write-coalescing via stream aggregation; never synchronous increment |
| Fanout worker crash mid-batch | Some followers never receive the tweet_id push | Consumer offset not committed | At-least-once delivery; idempotent push with a per-tweet "already pushed" set |
| Search indexing lag | New tweets not searchable for minutes | Consumer lag on indexer | Scale indexer fleet; apply backpressure rather than dropping events |
| Snowflake clock skew | Duplicate/out-of-order tweet IDs if a node's clock jumps backward | ID generation node detects `now < last_timestamp` | Refuse to generate IDs until clock catches up; NTP monitoring on all ID-gen nodes |
| Follow graph shard hot spot (celebrity followee list) | One shard serves disproportionate traffic | Per-shard QPS monitoring | Split celebrity follower lists across multiple sub-shards |
| Reply store hot partition (viral tweet's replies) | One partition serves disproportionate reply traffic | Per-tweet_id reply write/read rate | Same sub-partitioning pattern as a hot domain/celebrity — split by reply_id hash within the tweet_id |
| API Gateway overload / DDoS | Sudden traffic surge degrades all downstream services | Real-time alert from Prometheus/Alertmanager | IP-based rate limiting at the gateway; autoscaling; upstream WAF |

---

## 20. Scalability & Sharding

### General Sharding Strategy

```
Tweet Store:        shard by hash(user_id) — collocates a user's
                     own tweets for profile-timeline reads
Reply Store:         indexed/sharded by tweet_id
Follow Graph:        shard by hash(user_id), separate structures
                     for "following" vs "followers" direction
Timeline Cache:       shard by hash(user_id) across cache cluster
Message queue:        partition by hash(author_id) — preserves
                     per-author ordering for fanout workers
```

### Horizontal Scaling Plan

| Component | Scaling Strategy |
|---|---|
| API Gateway / stateless services | Add nodes behind the load balancer |
| Fanout workers | Consumer group; add workers, add partitions |
| Timeline cache | Cluster mode; consistent hashing across shards |
| Tweet / Reply store | Add nodes; consistent hash ring; replication factor 3 |
| Search (Elasticsearch) | Add data nodes; increase shard count per index |
| Ranking inference service | Stateless; horizontally scaled; can degrade to a simpler heuristic under load |

### Multi-Region Considerations

```
Write path: users write to nearest region; tweet replicated
            asynchronously to other regions (AP — eventual
            global consistency)
Read path:  timeline reads served from nearest region's cache;
            a user's own just-posted tweet must be read-your-
            own-writes consistent locally, even before cross-
            region replication completes
Fanout:     region-aware — a follower in another region gets
            the tweet pushed once cross-region replication
            delivers the source event
```

---

## 21. The Celebrity Problem (Hybrid Fanout Deep Dive)

This is the **single most important deep dive** in a Twitter system design interview.

### Why Pure Push Breaks

A celebrity with 100M followers posting a tweet triggers 100M timeline cache writes. At even modest tweet frequency, this alone can saturate the entire fanout pipeline and starve fanout for every other tweet in flight — a hot-key-induced systemic outage, not just a slow path for that one tweet.

### Why Pure Pull Breaks

If every timeline read merges live from every followee, a user following thousands of accounts triggers a large fan-in merge on every timeline load — this fails the low-latency requirement and creates massive read fan-in load on the Tweet Store for popular authors, whose tweets get re-fetched by every one of their followers' reads.

### The Hybrid Solution

```
Classification threshold (e.g. follower_count > 1M):

Regular accounts (< threshold):
  → PUSH: fanout-on-write into follower timeline caches
  → Timeline read: read precomputed cache — O(1)

Celebrity accounts (≥ threshold):
  → NO PUSH: tweet is NOT fanned out to followers' caches
  → Maintain a small, separately-cached "celebrity recent
    tweets" list per celebrity (one write regardless of
    follower count)
  → Timeline read: after reading the user's push-based cache,
    also fetch-and-merge recent tweets from each celebrity the
    user follows (bounded — most users follow only a handful of
    mega-accounts, so this merge is small)
```

### Why This Works

- Write cost for a celebrity tweet: **O(1)**, not O(followers)
- Read cost per user: bounded by **number of celebrities followed**, typically single digits
- The two expensive cases (many followers, many follows) never compound

### Threshold Tuning Is Itself a Tradeoff

A single fixed threshold is a Senior-level answer. Staff-level: the threshold should be **dynamic**, based on observed fanout cost — e.g., estimated fanout time or recent tweet frequency × follower count, re-evaluated periodically, so an account crossing 1M followers doesn't need manual reclassification, and a moderately-followed but extremely frequent poster (e.g., a news bot) can also be routed to pull even below the raw follower threshold.

### Interview Framing

State the problem before being asked: *"Naive push fanout breaks down for high-follower accounts — I'd classify accounts by follower count and use a hybrid push/pull strategy, precomputing timelines for regular accounts and merging celebrity tweets at read time."* Said proactively, this is often the highest-signal moment of the entire interview.

---

## 22. Security

No system design interview is complete without addressing security explicitly — plan for a few minutes near the end.

### Authentication & Authorization

Every request must be verifiably from a legitimate user, and that user must hold the right permissions for the action requested. Handled by the Auth Service (Section 6); the API Gateway short-circuits unauthenticated/unauthorized requests before they reach downstream services.

### Data Encryption

- **In transit:** HTTPS everywhere, ensuring data exchanged between client and servers is confidential and tamper-proof
- **At rest:** most modern databases and object stores support native encryption-at-rest — typically a configuration flag rather than custom engineering

### Rate Limiting

Two distinct layers, each solving a different problem:
- **Account-based** (Tweet/Reply Service): caps how many tweets/replies a single user can create in a period, preventing bot-driven flooding — see Section 7
- **IP-based** (API Gateway): caps requests per IP address within a time window, primarily to prevent DDoS attacks rather than to police legitimate individual users

### Input Validation

Validate and sanitize all user input on **both** client and server to prevent SQL injection, cross-site scripting (XSS), and similar attacks. Client-side validation is a UX nicety; server-side validation is the actual security boundary — never rely on the client alone.

---

## 23. Observability: Monitoring, Logging & Alerting

### System Health Checks

Every service (Tweet CRUD, Reply, Profile, Auth, Timeline) needs continuous health monitoring so an outage or degraded-latency condition is caught immediately rather than discovered via user complaints. **Prometheus** for metrics collection, **Grafana** for visualization dashboards, is the standard combination to name here.

### Logging

Every significant action — a tweet posted, a login, a failed auth attempt — should be logged, both for debugging and for tracking potential security threats. The **ELK stack** (**E**lasticsearch for storage, **L**ogstash for processing/ingestion, **K**ibana for visualization) is the standard answer: every service, including the API Gateway and the databases themselves, ships logs to this central pipeline.

### Real-Time Alerting

A sudden traffic surge, an unusual spike in failed logins, or a service health check failure should trigger an immediate notification rather than waiting to be noticed on a dashboard. **Alertmanager** or **PagerDuty**, integrated with Prometheus, routes these alerts to email/Slack/on-call paging so the team can respond quickly.

---

## 24. Testing & CI/CD

### Load Testing

Before shipping any new feature, validate how the system — particularly high-traffic services like Tweet CRUD or Profile — holds up under increased load. This surfaces bottlenecks and potential failure points before they hit production.

### Automated Testing

Given the microservice architecture, seamless integration between services is critical on every change. CI tools (**Jenkins**, **GitHub Actions**) automatically run both **unit tests** (individual components in isolation) and **integration tests** (verifying services communicate correctly) on every code change.

### Backup & Recovery Testing

User data is invaluable — regular backups are non-negotiable, but a backup you've never tested restoring from is not a real safety net. Periodically test the actual recovery process so that, in the event of a real failure, the system can be restored swiftly and predictably rather than discovering gaps in the recovery plan during an actual incident.

---

## 25. Senior vs Staff Answer Differentiators

### Senior-level expectations

- Identify core components: clients, load balancer, API gateway, tweet/reply/profile services
- Explain basic fanout-on-write model
- Mention a message queue as the backbone
- Basic capacity math (tweets/sec, reads/sec)
- Cover security, monitoring, and testing at a checklist level when prompted

### Staff-level expectations (additional)

| Topic | What Staff-level adds |
|---|---|
| Fanout strategy | Proactively identify the celebrity problem and design the hybrid push/pull model with a dynamic threshold |
| Service boundaries | Justify *why* replies, auth, and profile are split into separate services — tie each split to a specific scaling or security reason, not just "microservices are good practice" |
| Tweet ID generation | Snowflake ID structure and why auto-increment/UUID fail here |
| Counters | Write-coalescing via stream aggregation instead of synchronous row/document increments |
| CAP theorem | Explicit AP-by-default reasoning tied to product semantics, with named CP exceptions (auth, profile integrity, abuse counters) and *why* they differ |
| Caching architecture | Explicit separation of ID-list timeline cache vs. tweet-content cache vs. CDN edge cache, and why that separation matters |
| Search | CDC-based index sync instead of a risky dual write |
| Trending topics | Rate-of-change / spike detection, not raw count; HyperLogLog for unique participant estimation |
| Notification fanout | Recognize it as structurally the same fan-out problem as timelines, needing the same aggregation strategy |
| Multi-region | Read-your-own-writes locally vs. async cross-region replication for everything else |
| Dormant users | Lazy/deferred fanout for inactive accounts to avoid wasted push work |
| Security/observability/testing | Treat these as integral design decisions with specific tool tradeoffs, not a generic closing checklist |

### The single most important Staff differentiator

**Recognizing that Twitter is fundamentally a fanout/amplification problem, not a storage problem**, and designing the hybrid push/pull fanout strategy before the interviewer has to lead you there. If you only describe pure fanout-on-write and wait for the interviewer to ask "what about someone with 100 million followers?", you're presenting at Senior level.

---

## 26. Interview Time Allocation

For a 45-minute system design session:

| Phase | Time | Focus |
|---|---|---|
| Requirements & scope | ≤5 min | Confirm functional + non-functional; explicitly bound scope (no DMs, no ads, no full recommendation engine) |
| High-level architecture | 5 min | Client → LB → API Gateway → services; name the message queue as backbone |
| Capacity estimation | 5 min | Tweets/sec, reads/sec, read:write ratio, fanout amplification |
| Tweet/Reply/Profile services | 8 min | Data model choices (document vs wide-column, SQL vs graph DB) with justification |
| Timeline / Fanout (deep dive) | 12 min | Fanout-on-read baseline → fanout-on-write → hybrid model → celebrity problem in full |
| Search / Trending / Notifications | 4 min | Brief on each; don't over-invest unless prompted |
| Security, monitoring, testing | 5 min | Hit each briefly with named tools and specific reasoning — don't skip this closing section |

**What to cut if short on time:** Search and trending topics detail; go briefer on security/monitoring/testing tool names if truly pressed. **Never cut:** the fanout strategy and celebrity problem.

---

## 27. Quick-Reference Cheatsheet

```
KEY ALGORITHMS
──────────────
Tweet ID:        Snowflake (timestamp + machine ID + sequence) — coordination-free
Fanout:          Hybrid push/pull, dynamic follower-count threshold
Timeline merge:  K-way merge of push results + pull results, then rank
Counters:        Stream write-coalescing, not sync row/document updates
Trending:        Sliding-window rate-of-change / spike detection + HyperLogLog
Search sync:     Change Data Capture (CDC), not dual writes

KEY NUMBERS
───────────
300M DAU  →  ~5,800 tweets/sec avg write, ~17,400 peak
~34,700 timeline reads/sec avg, ~104,000 peak
Read:write ratio ≈ 6:1 at request level; far higher fanout-adjusted
Avg 700 followers/author; celebrity threshold ~1M followers
~1.9 TB for timeline ID caches (800 IDs × 300M users)
~270 TB tweet metadata over 5 years (excludes media)

KEY SYSTEMS
───────────
Entry point:       Client → L7 Load Balancer (round robin) → API Gateway
Message bus:        Kafka / message queue (new-tweets, engagement-events)
Tweet store:         MongoDB (document) or Cassandra (wide-column) — justify the pick
Reply store:          separate store, indexed by tweet_id
Timeline cache:      Redis/Memcached, list of tweet_ids per user, capped ~800
Content cache:       tweet_id → hydrated content, reused across followers
Follow graph:         Graph DB (multi-hop features) or sharded KV (single-hop only)
User profiles:        SQL (structured, ACID)
Search:              Elasticsearch, synced via CDC
Media:               S3/Blob storage + CDN
Trending:            Flink/Kafka Streams + sorted-set store
Monitoring:          Prometheus + Grafana
Logging:             ELK stack (Elasticsearch, Logstash, Kibana)
Alerting:            Alertmanager / PagerDuty
CI/CD:               Jenkins / GitHub Actions — unit + integration tests

CAP DECISIONS
─────────────
AP (default):  Tweet/reply store, timeline cache, content cache, counters,
               search index, follow-graph propagation to fanout
CP (exceptions): Auth/session store, user profile integrity (SQL),
               rate-limiting/abuse counters

THE CELEBRITY PROBLEM — CORE ANSWER
────────────────────────────────────
Regular accounts:   push (fanout-on-write) → O(1) timeline reads
Celebrity accounts: no push → merged at read time → O(1) tweet writes
Threshold:          dynamic, based on follower count × post frequency
Result:             decouples write cost from follower count and
                     read cost from following count

SECURITY / OBSERVABILITY / TESTING — DON'T SKIP
─────────────────────────────────────────────────
Security:     AuthN/AuthZ, HTTPS + at-rest encryption, dual-layer
              rate limiting (account + IP), input validation
Monitoring:   Prometheus/Grafana health checks, ELK logging,
              Alertmanager/PagerDuty real-time alerts
Testing:      Load testing, CI unit + integration tests,
              backup-and-recovery drills (not just backups)

STAFF-LEVEL SIGNALS TO HIT
───────────────────────────
✓ Hybrid push/pull fanout with the celebrity problem stated proactively
✓ Snowflake ID generation and why naive alternatives fail
✓ Justified datastore choices per service (document vs wide-column,
  SQL vs graph DB) rather than defaulting to one DB everywhere
✓ Separate Reply Service with clear scalability/read-perf reasoning
✓ CDC for search index sync instead of risky dual writes
✓ Write-coalescing for hot counters instead of row-level increments
✓ AP-by-default reasoning tied to product semantics, with named CP exceptions
✓ Dormant-user lazy fanout to avoid wasted push work
✓ Security/monitoring/testing treated as designed tradeoffs, not a checklist
✓ Proactive failure mode enumeration (hot partitions, cache stampedes, fanout storms)
```

---

*Guide compiled for Senior & Staff-level system design interview preparation.*
*Topics: Twitter, Distributed Systems, Microservices, Feed Fanout, CAP Theorem, Security, Observability.*
