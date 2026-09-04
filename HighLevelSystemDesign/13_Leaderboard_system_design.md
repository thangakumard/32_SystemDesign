# Leaderboard System — System Design Interview Guide
### Senior & Staff Engineer Level

---

## Table of Contents

1. [Problem Statement & Scope](#1-problem-statement--scope)
2. [Requirements](#2-requirements)
3. [Capacity Estimation](#3-capacity-estimation)
4. [High-Level Architecture](#4-high-level-architecture)
5. [Component 1: Score Ingestion API](#5-component-1-score-ingestion-api)
6. [Component 2: Event Bus (Kafka)](#6-component-2-event-bus-kafka)
7. [Component 3: Ranking Engine — Sharded Sorted Sets](#7-component-3-ranking-engine--sharded-sorted-sets)
8. [Component 4: Rank Aggregation Service](#8-component-4-rank-aggregation-service)
9. [Component 5: Persistent Score Store](#9-component-5-persistent-score-store)
10. [Component 6: Read API & Caching Layer](#10-component-6-read-api--caching-layer)
11. [Component 7: Real-Time Push Service](#11-component-7-real-time-push-service)
12. [Component 8: Time-Windowed & Scoped Leaderboards](#12-component-8-time-windowed--scoped-leaderboards)
13. [Component 9: Tie-Breaking & Anti-Cheat](#13-component-9-tie-breaking--anti-cheat)
14. [CAP / PACELC Positioning](#14-cap--pacelc-positioning)
15. [Failure Modes & Mitigations](#15-failure-modes--mitigations)
16. [Scalability & Sharding](#16-scalability--sharding)
17. [Consistency Models: Exact vs. Approximate Rank](#17-consistency-models-exact-vs-approximate-rank)
18. [Senior vs Staff Answer Differentiators](#18-senior-vs-staff-answer-differentiators)
19. [Interview Time Allocation](#19-interview-time-allocation)
20. [Quick-Reference Cheatsheet](#20-quick-reference-cheatsheet)

---

## 1. Problem Statement & Scope

A **leaderboard system** maintains a ranked ordering of entities (players, users, teams) by score, and answers three query shapes in real time:

1. **Top-N** — "show me the top 100 players"
2. **Rank-of-entity** — "what is my current rank?"
3. **Range-around-entity** — "show me the 5 players above and below me"

Scores update continuously (game points, workout minutes, sales numbers, coding-contest points), and the system must reflect updates with low latency while serving read-heavy traffic at very high QPS — leaderboard screens are checked far more often than scores are submitted.

### What the interviewer is really testing

- Do you know that a **naive `ORDER BY score` on a relational table collapses under real-time write/read load**, and reach for the right data structure (sorted set) instead?
- Can you reason about what happens **when a single sorted set no longer fits on one node** — this is the crux of the problem, and most candidates never get there?
- Do you understand the **exact-vs-approximate rank tradeoff** at extreme scale?
- Can you design for **multiple simultaneous leaderboard scopes** (global, regional, friends, per-game-mode, per-time-window) without re-deriving the architecture for each one?
- Do you proactively address **tie-breaking, anti-cheat, and hot-key problems**?

---

## 2. Requirements

### Functional Requirements

| Requirement | Notes |
|---|---|
| Submit/update a score for an entity | Absolute score or delta increment |
| Get Top-N leaderboard | N typically 10–100 |
| Get an entity's current rank and score | Point lookup |
| Get entities "around" a given entity | Neighbors above/below by rank |
| Support multiple scopes | Global, regional, friends-only, guild, per-game-mode |
| Support multiple time windows | Daily, weekly, monthly, all-time, rolling N-day |
| Deterministic tie-breaking | Two equal scores must still produce a strict order |

### Non-Functional Requirements

| Requirement | Target |
|---|---|
| Scale | 100M+ monthly active entities; hundreds of millions of score events/day |
| Read latency | p99 < 100ms for Top-N and rank lookup |
| Write-to-visible latency | Score change reflected within 1–5 seconds |
| Read:write ratio | Typically 10:1 to 50:1 — leaderboards are read-dominated |
| Availability | Leaderboard is often core UX; degrade to stale data, never hard-fail |
| Consistency | Exact rank for small/scoped leaderboards; approximate acceptable for global rank at extreme scale |

---

## 3. Capacity Estimation

```
Assume: competitive game or platform, 100M MAU, 20M DAU

Score events:
  20M DAU × 15 score-affecting actions/day ≈ 300M events/day
  300,000,000 / 86,400 ≈ 3,472 events/sec average
  Peak (tournament / prime-time, 10×) ≈ 35,000 events/sec

Read traffic (20:1 read:write ratio):
  3,472 × 20 ≈ 69,000 reads/sec average
  Peak ≈ 690,000 reads/sec  →  caching is not optional, it's structural

Sorted-set write throughput per Redis node:
  Single-threaded command execution ≈ 50,000–100,000 ops/sec ceiling
  Realistic sustained throughput with network/serialization overhead: ~20,000–40,000 ops/sec
  → Peak of 35,000 events/sec already saturates a single node
  → Minimum viable shard count: 4–8, sized for 5–10× future headroom: 16–32 shards

Memory footprint (single all-time global leaderboard):
  100M entries × ~80 bytes/entry (member ID + score + skip-list pointers)
  ≈ 8 GB — trivial for a Redis cluster on its own

The real memory driver — leaderboard multiplication:
  scopes (global, 5 regions, friends, guild) × time windows (daily, weekly,
  monthly, all-time) × game modes (10) ≈ 100s of independent sorted sets
  Each is far smaller than the full 100M (only active-in-window entities),
  but the multiplication is why this is a sharding and lifecycle problem,
  not a "will it fit in RAM" problem.
```

**Key insight:** Unlike the web crawler (where storage dominates), a leaderboard's dominant cost is **write throughput concentrated on a small number of hot keys**, multiplied across many leaderboard scopes. Raw data volume is rarely the bottleneck — contention is.

---

## 4. High-Level Architecture

```
                    ┌───────────────────────┐
                    │   Score Producers      │
                    │  (game clients/servers)│
                    └───────────┬────────────┘
                                │
                    ┌───────────▼────────────┐
                    │  Score Ingestion API    │
                    │  • auth + rate limit    │
                    │  • server-side validate │
                    │  • idempotency key      │
                    └───────────┬────────────┘
                                │  (Kafka topic: score-events)
                    ┌───────────▼────────────┐
                    │      Event Bus          │
                    │       (Kafka)           │
                    │  partitioned by entity  │
                    └──────┬──────────┬───────┘
                           │          │
          ┌─────────────────┘          └────────────────┐
          │                                             │
┌─────────▼──────────┐                     ┌─────────────▼─────────────┐
│  Ranking Engine     │                     │  Persistent Score Store   │
│  Sharded Sorted     │                     │  Cassandra / DynamoDB     │
│  Sets (Redis)       │                     │  (source of truth,        │
│  Shard 0..N         │                     │   rebuild-from-log)       │
└─────────┬──────────┘                     └───────────────────────────┘
          │
┌─────────▼──────────────┐
│  Rank Aggregation       │
│  Service                │
│  (fan-out across shards)│
└─────────┬──────────────┘
          │
┌─────────▼──────────┐        ┌────────────────────────┐
│  Read API            │──────▶│  Cache Layer            │
│  Top-N / Rank / Range│        │  (Top-N, hot ranks)     │
└─────────┬───────────┘        └────────────────────────┘
          │
┌─────────▼──────────────┐
│  Real-Time Push Service │
│  (WebSocket / SSE)      │
│  rank-change notify     │
└─────────────────────────┘
```

**Kafka sits at the center**, exactly as in a crawler pipeline: it decouples ingestion from ranking, provides replay (rebuild a lost Redis shard from the event log), and lets multiple independent consumers (daily leaderboard, weekly leaderboard, analytics, anti-cheat) all read the same durable stream. See the *Apache Kafka* guide for partition/consumer-group mechanics that apply directly here.

---

## 5. Component 1: Score Ingestion API

### Responsibilities

```
1. Authenticate the submitting entity (never trust client-reported scores blindly)
2. Rate-limit per entity (prevent submission flooding)
3. Validate score plausibility (see Component 9: Anti-Cheat)
4. Attach a globally unique, monotonically-informative idempotency key
5. Publish to Kafka topic: score-events
6. Return 202 Accepted — do NOT block on ranking engine update
```

### Why the API must not block on the ranking write

Coupling ingestion to a synchronous Redis write means an ingestion-path outage cascades into a ranking-engine outage, and vice versa. Publishing to Kafka and returning immediately gives:

- **Write absorption** during traffic spikes (tournament finales, viral moments) — Kafka buffers what Redis can't yet drain
- **Independent scaling** of ingestion vs. ranking
- **Replay** capability if the ranking engine needs to be rebuilt

### Idempotency

At-least-once delivery (Kafka's default durability guarantee) means the same score event can be processed twice. Two idempotency strategies:

| Strategy | Mechanism | When to use |
|---|---|---|
| Absolute score writes | `ZADD` with the entity's new total score — naturally idempotent, replay is a no-op | Score is a running total the client/server always knows |
| Delta writes with dedup | Attach `event_id`; ranking worker checks a short-TTL dedup set (Redis `SET NX`) before applying | Score is computed as increments (e.g., "+10 points this kill") |

Absolute-score writes are strictly preferable when available — they sidestep an entire class of duplicate-delivery bugs. See the *Distributed Unique ID Generation* guide for `event_id` generation approaches.

---

## 6. Component 2: Event Bus (Kafka)

### Partitioning Strategy

```
Partition key = entity_id (hash)

Why: preserves per-entity ordering. If two score updates for the same
entity arrive out of order, the ranking engine could apply them in the
wrong sequence and leave a stale score visible. Hashing on entity_id
guarantees all events for one entity land on the same partition, and
Kafka guarantees in-order delivery within a partition.
```

### Topics

| Topic | Purpose | Retention |
|---|---|---|
| `score-events` | Raw validated score submissions | 7 days (enables ranking-engine rebuild) |
| `rank-changes` | Derived events: "entity X moved from rank M to rank N" | 24 hours (feeds push service) |
| `anti-cheat-flags` | Suspicious submissions routed for review | 30 days |

### Why This Matters at Staff Level

The event bus isn't just a queue — it's the **system of record for score history** and the mechanism that lets Redis be treated as a disposable, rebuildable cache rather than a fragile source of truth. Losing a Redis shard becomes an operational non-event: replay `score-events` from the last checkpoint.

---

## 7. Component 3: Ranking Engine — Sharded Sorted Sets

This is the heart of the system — the component most candidates under-design.

### The Base Data Structure: Sorted Set

A **sorted set** (Redis `ZSET`, backed by a skip list + hash table) gives:

```
ZADD leaderboard:global 4820 "user_42"      → O(log N) insert/update
ZREVRANK leaderboard:global "user_42"       → O(log N) rank lookup
ZREVRANGE leaderboard:global 0 99           → O(log N + M) top-N (M=100)
ZREVRANGE leaderboard:global rank-5 rank+5  → O(log N + M) range-around-entity
```

This single structure natively answers all three core query shapes with logarithmic cost. It is the **single most important building block in this entire design** — but it only works as long as the whole leaderboard fits on one node and one thread.

### The Scaling Wall

A single Redis instance is single-threaded per command and memory-bound to one machine. At tens of thousands of writes/sec sustained (Section 3), one `ZSET` becomes both a **write bottleneck** (single-threaded command execution) and a **single point of failure** for the entire leaderboard. This is the point where the "textbook Redis ZSET leaderboard" answer stops being sufficient — and where the Staff-level design diverges from the Senior-level one.

### Sharding Strategy A: Shard by Entity ID Hash

```
shard_id = hash(entity_id) % num_shards

Shard 0: ZSET containing entities where hash % N == 0
Shard 1: ZSET containing entities where hash % N == 1
...
```

| Pros | Cons |
|---|---|
| Even write distribution — no hot shard regardless of score distribution | **No shard has global rank context** — computing "what is my global rank" requires querying every shard for "how many entities have a higher score than me" and summing |
| Simple, well-understood consistent-hashing story | "Range around me" is expensive — neighbors in global rank are scattered across shards |
| Easy to add shards with standard rebalancing | Top-N requires a full fan-out + merge across all shards on every query (unless cached) |

### Sharding Strategy B: Shard by Score Range (Bucketing)

```
Shard 0: scores [0        – 10,000)
Shard 1: scores [10,000   – 25,000)
Shard 2: scores [25,000   – 50,000)
Shard 3: scores [50,000   – 100,000)
...
Shard N: scores [X        – ∞)
```

| Pros | Cons |
|---|---|
| Global rank = `sum(count of higher-scoring shards) + local rank within your shard` — O(shards) fan-out instead of O(all entities) | **Hot-shard risk**: if scores cluster (e.g., competitive ranked play converges near a skill ceiling), one bucket absorbs disproportionate write traffic |
| Top-N is trivial — always lives entirely in the highest shard | Bucket boundaries need periodic rebalancing as the score distribution shifts over a season |
| "Range around me" mostly stays within one shard (occasionally spans a boundary) | Boundary migrations are operationally delicate — an entity's score crossing a boundary means moving it between shards |

### Which to Choose (Staff-Level Reasoning)

```
Choose by-entity-hash sharding when:
  • Write throughput is the primary risk (huge concurrent player base)
  • Global exact rank is rarely queried on the hot path (only Top-N and
    "my rank" are needed, and both can be served from a cache/aggregator)

Choose by-score-range sharding when:
  • Rank and range-around-me queries are the dominant read pattern
    (competitive ladders where "who's near me" is core UX)
  • Score distribution is reasonably spread (not everyone converging to
    one narrow band) — or you're willing to operate dynamic rebalancing

Hybrid (what most large-scale systems actually run):
  • By-entity-hash shards for the write-heavy raw ranking engine
  • A separately materialized, periodically rebuilt score-range index
    for cheap rank/range queries — trading some staleness for cheap reads
```

This shard-strategy choice — and its direct consequence for which query becomes expensive — is the **single most important Staff-level differentiator** in this whole guide. See the *Sharding Strategies* guide for the general hash-vs-range tradeoff this specializes.

---

## 8. Component 4: Rank Aggregation Service

Sits between the sharded ranking engine and the Read API. Its job depends on the sharding strategy chosen above.

### For Entity-Hash Sharding (fan-out and merge)

```
To compute global Top-N:
  1. Query ZREVRANGE 0 N on every shard in parallel
  2. Merge N×shard_count candidates with a min-heap → true global Top-N
  3. Cache result (Top-N changes slowly relative to total write volume)

To compute an entity's global rank:
  1. Look up entity's own score (O(1) — you know which shard it's on)
  2. Query every OTHER shard: "how many entities have score > X?"
     (ZCOUNT with a range query, O(log N) per shard)
  3. Sum counts across shards + local rank within own shard = global rank
  4. Fan-out is parallel; p99 latency ≈ slowest shard, not sum of shards
```

### For Score-Range Sharding (prefix-sum)

```
To compute an entity's global rank:
  1. Identify entity's shard (its score falls in a known bucket range)
  2. Sum the total member count of every shard ABOVE this one
     (maintain a running per-shard count, updated on writes — O(1) read)
  3. Add local rank within own shard (ZREVRANK, O(log N))
  4. No fan-out needed for the "higher shards" count — just cached counters
```

### Staff-Level Gotcha: Fan-Out Timeout Handling

A single slow or partitioned shard should never block the entire rank computation. Set an aggressive per-shard timeout (e.g., 50ms) and:

```
On shard timeout:
  • Fall back to the last cached count for that shard (slightly stale, bounded)
  • Never fail the whole request for one slow shard
  • Surface a "rank may be approximate" flag internally for monitoring
```

This is the same principle as the crawler's per-domain politeness isolation: **one degraded partition should degrade precision, not availability.**

---

## 9. Component 5: Persistent Score Store

Redis holds the *operational* ranking structure — fast, but treated as rebuildable. A separate durable store holds the source-of-truth score history.

| Layer | System | Rationale |
|---|---|---|
| Ranking engine | Redis Cluster (sharded ZSETs) | Sub-ms rank/range operations; the only structure that answers rank queries cheaply |
| Score history / source of truth | Cassandra / DynamoDB | Durable, high write throughput, partition by `entity_id`; survives Redis data loss |
| Rebuild path | Kafka `score-events` replay | If a Redis shard is lost, replay events (or bulk-load from Cassandra) to reconstruct it |

```sql
-- Cassandra schema (source of truth)
CREATE TABLE entity_scores (
  entity_id       TEXT,
  leaderboard_id  TEXT,      -- scope: global / region / mode / window
  score           BIGINT,
  updated_at      TIMESTAMP,
  event_id        TEXT,      -- idempotency key
  PRIMARY KEY (leaderboard_id, entity_id)
);
```

Redis is never the only copy of the data. This is the same "durable log + disposable fast-path index" pattern used for the crawler's dedup Bloom filter and frontier state, generalized here to ranking.

---

## 10. Component 6: Read API & Caching Layer

### Query Shapes and Their Cache Strategy

| Query | Cache strategy | TTL / invalidation |
|---|---|---|
| Top-N (global) | Cache the full Top-N result; changes slowly relative to total write volume | Refresh every 1–5s, or invalidate on any write that could enter the Top-N band |
| Rank-of-entity | Cache per-entity, short TTL | Refresh every 5–10s; acceptable staleness for most UX |
| Range-around-entity | Rarely cacheable (personalized per requester) — served live from aggregation service | N/A — optimize the aggregation path instead |

### The Celebrity / Hot-Key Read Problem

A famous streamer's rank or profile can receive read volume orders of magnitude above a typical entity — the read-path mirror of the crawler's hot-domain write problem. Mitigation:

- **Per-entity read cache** with request coalescing (single-flight): concurrent requests for the same hot entity share one upstream lookup instead of stampeding the aggregation service
- **Cache stampede protection** on Top-N refresh: use a short lock or stale-while-revalidate pattern so a cache expiry under peak load doesn't trigger a thundering herd of aggregation fan-outs simultaneously

---

## 11. Component 7: Real-Time Push Service

Many leaderboard UIs want live rank-change notifications ("you just passed 3 players!") without polling.

```
Ranking engine write
        │
        ▼
Emit to Kafka topic: rank-changes
  { entity_id, old_rank, new_rank, leaderboard_id, timestamp }
        │
        ▼
Push Service (WebSocket/SSE gateway)
  • Subscribes to rank-changes for entities with an active connection
  • Only computes/pushes deltas for entities actually being watched —
    do NOT push every score event to every connected client
```

**Staff-level nuance:** compute rank *deltas*, not raw score events, and only for subscribed entities. Pushing every raw score update to every connected client turns a write-heavy backend problem into a fan-out messaging problem at a much larger multiplier (millions of concurrent viewers × every score tick).

---

## 12. Component 8: Time-Windowed & Scoped Leaderboards

### Fixed Windows (Daily/Weekly/Monthly)

```
Key pattern: leaderboard:{scope}:{window}:{date_bucket}
Example:     leaderboard:global:daily:2026-09-03

• Each window is an independent sorted set
• Expire/archive old windows via TTL or scheduled compaction
• "All-time" is simply the window with no expiry
```

This is cheap: a new day just means a new key, no need to remove old entries from an existing structure.

### Rolling Windows (Trailing 7 Days) — The Harder Case

A single expiring key doesn't work for "trailing N days," because entries need to age out continuously, not all at once at a boundary.

```
Approach 1 — Time-decay scoring:
  effective_score = raw_score × decay_factor^(days_since_earned)
  Re-score periodically (or compute decay at read time).
  Approximate; no exact "last 7 days only" guarantee.

Approach 2 — Per-day buckets + merge:
  Maintain one sorted set per day (7 of them for a 7-day window).
  Query time: merge the 7 sets (sum scores per entity across them).
  Exact, but read-time cost scales with window length — cache the merge.

Approach 3 — Approximate sliding structures:
  For read-heavy, precision-tolerant use cases, use a decaying counter
  (e.g., exponential moving average) per entity instead of true windowed
  sums. Cheapest at massive scale; least precise.
```

**Staff-level answer:** name the exactness/cost tradeoff explicitly and pick per-day-bucket merge as the default (exact, bounded cost, cacheable), reserving decay approximations for extreme-scale rolling windows where merge cost becomes prohibitive.

### Scope Multiplication

```
leaderboard:{region}:{mode}:{window}:{date_bucket}

100 MAU-scale platform example:
  5 regions × 10 game modes × 4 windows ≈ 200 independent leaderboards

Each is sharded independently only once it individually exceeds a
single shard's capacity — don't pre-shard small leaderboards (e.g.,
a friends-only leaderboard with 50 entries needs exactly one ZSET key,
no sharding infrastructure at all).
```

---

## 13. Component 9: Tie-Breaking & Anti-Cheat

### Deterministic Tie-Breaking

Two entities with the identical score need a stable, fair secondary order — typically "whoever reached this score first."

```
Naive approach: separate lookup for tiebreak → extra round trip, race conditions

Staff approach: encode tiebreak into the sort key itself
  composite_score = (raw_score × 10^13) - timestamp_reached_ms

  • Higher raw_score always dominates (multiplier keeps digits separate)
  • Among equal raw_score, EARLIER timestamp produces a LARGER composite
    value (since it's subtracted) → earlier achiever ranks higher
  • Single ZADD, single sort dimension — no secondary lookup needed

Use a server-assigned monotonic timestamp/sequence number (never the
client's clock) — see Distributed Unique ID Generation guide for
sequence generation approaches that avoid clock-skew tie-break bugs.
```

### Anti-Cheat as a First-Class Concern

Client-reported scores are an adversarial input by default.

```
Ingestion-time checks:
  • Rate limiting per entity (implausible submission frequency)
  • Bounds checking (score delta exceeds physically possible max for
    the time elapsed since last submission)
  • Server-authoritative scoring where feasible (compute score server-side
    from game-state events, never trust a raw client-submitted number)

Post-hoc checks (async, off the hot path):
  • Statistical anomaly detection (z-score outliers vs. population,
    sudden jumps inconsistent with skill progression)
  • Shadow-ban / flag-for-review rather than silent rejection, to avoid
    tipping off cheaters while investigation is pending
  • Route flagged events to the anti-cheat-flags Kafka topic (Component 6)
    for asynchronous review — never block the ingestion hot path on this
```

---

## 14. CAP / PACELC Positioning

At Staff level, address CAP **and** the PACELC "else" branch — because for the ranking engine, the more interesting tradeoff isn't partition behavior, it's the everyday **latency-vs-consistency** choice made on every rank query, partition or not.

| Component | CAP Choice | PACELC (no-partition) | Reasoning |
|---|---|---|---|
| Score Ingestion API / Kafka | **AP** (durability-favoring) | Else: Latency favored — async publish, don't block on downstream | Absorbing writes matters more than immediate ranking visibility |
| Ranking Engine — single shard | **CP** within the shard | Else: Consistent — single-threaded Redis makes per-shard operations effectively linearizable | Local shard state is always internally consistent |
| Ranking Engine — global aggregate rank | **AP** across shards | Else: **Latency favored by default** — exact global rank requires synchronous fan-out to every shard; most systems cap fan-out with timeouts and accept a slightly stale/approximate rank instead | This is the core Staff-level tradeoff: exact rank costs latency, cheap rank costs precision |
| Persistent Score Store (Cassandra) | **AP** | Else: tunable — `QUORUM` reads for reconciliation jobs, `ONE` for high-throughput ingestion writes | Standard wide-column tunable consistency |
| Top-N Cache | **AP** | Else: Latency favored — serve slightly stale Top-N rather than compute fresh on every request | Top-N changes slowly; staleness is cheap, recomputation is not |
| Real-Time Push Service | **AP** | Else: Latency favored — best-effort delivery; a missed rank-change notification self-corrects on the next update | Notifications are a UX nicety, not a correctness requirement |

**The one line that matters most:** *exact global rank at massive scale is a synchronous fan-out problem, and every leaderboard system eventually chooses to trade some consistency for latency on that specific query — the design question is where you draw that line, not whether you draw it.*

Cross-reference the *CAP / PACELC Theorem* guide for the general framework this specializes.

---

## 15. Failure Modes & Mitigations

| Failure | Symptom | Detection | Mitigation |
|---|---|---|---|
| Redis shard node failure | Leaderboard shard unavailable; rank queries touching that shard fail | Replica promotion lag; error rate spike on affected shard | Redis Cluster with replicas + automated failover; rebuild from Kafka replay if data lost |
| Score-range shard hotspot | One bucket absorbs disproportionate write traffic (score convergence) | Per-shard write-rate monitoring shows skew | Dynamic bucket rebalancing; split hot bucket into sub-buckets |
| Fan-out slow-shard straggler | Rank query latency spikes; p99 dominated by slowest shard | Per-shard latency histograms in aggregation service | Aggressive per-shard timeout + fallback to cached count for that shard |
| Cache stampede on Top-N expiry | Cache miss under peak load triggers simultaneous full re-aggregation | Aggregation service request rate spikes in sync with cache TTL | Stale-while-revalidate; single-flight lock so only one request recomputes |
| Duplicate score event (at-least-once delivery) | Score double-counted on delta-based updates | Score jumps inconsistent with expected event size | Idempotency key + dedup window; prefer absolute-score writes over deltas |
| Client clock skew in tie-break timestamp | Unfair tie-break ordering; disputes over rank | Tie-break timestamps outside plausible server-time bounds | Server-assigned monotonic sequence number, never client-supplied time |
| Celebrity/hot-entity read hotspot | Single entity's rank lookup overwhelms aggregation service | Per-entity read-rate outlier detection | Per-entity cache with request coalescing (single-flight) |
| Kafka consumer lag (ranking workers fall behind) | Score updates visible with growing delay | Consumer lag monitoring on `score-events` | Scale ranking-worker fleet; increase partitions; backpressure alerts |
| Cheating / score manipulation | Implausible score spikes; leaderboard integrity complaints | Statistical anomaly detection; rate-limit violations | Server-authoritative scoring where possible; async flag-for-review pipeline |
| Bucket boundary migration (score-range resharding) | Entity's score crosses a shard boundary mid-season | Boundary-crossing event count per rebalance | Dual-write during migration window; atomic cutover with verification |

---

## 16. Scalability & Sharding

### Ranking Engine Sharding (recap with rebalancing detail)

```
Entity-hash shards:
  Standard consistent hashing; adding a shard triggers a bounded
  re-mapping of entities, same as any consistent-hash-ring resize.
  See Sharding Strategies guide for the general mechanics.

Score-range shards:
  Rebalancing means moving a contiguous score band's worth of entities
  to a new/adjacent shard. Do this as an online migration:
    1. New shard boundary declared; writes dual-written to old+new shard
       for entities in the transition band
    2. Backfill existing entities in that band from old shard to new
    3. Verify counts match; cut reads over to new shard
    4. Stop dual-write; decommission old shard's copy of that band
```

### Horizontal Scaling Plan

| Component | Scaling Strategy |
|---|---|
| Ranking engine (Redis) | Add shards; consistent hash or dynamic score-range rebalancing |
| Rank aggregation service | Stateless; add nodes behind the fan-out layer |
| Persistent score store (Cassandra) | Add nodes; consistent hash ring; replication factor 3 |
| Kafka | Add partitions (bounded by desired ordering granularity = entity_id) |
| Cache layer | Add nodes; partition by leaderboard/entity key |
| Push service | Add gateway nodes; shard WebSocket connections by entity subscription hash |

### Backpressure

```
Monitor: ranking_worker_lag = latest_offset - committed_offset (per partition)

If ranking_worker_lag > threshold:
  → Scale up ranking-worker fleet consuming score-events
  → Widen acceptable rank-staleness bound temporarily (communicate via
    the aggregation service's "approximate" flag) rather than blocking
    ingestion
  → Alert on-call if lag exceeds SLO
```

---

## 17. Consistency Models: Exact vs. Approximate Rank

At small-to-medium scale (friends leaderboard, guild leaderboard, single-shard global leaderboard), **exact rank is cheap and there's no reason to give it up.** The approximate-rank conversation only becomes necessary at the extreme end — hundreds of millions of concurrently ranked entities with high write concurrency.

| Approach | Precision | Cost | When to use |
|---|---|---|---|
| Single sorted set | Exact | Cheap until it isn't (single-node ceiling) | Small/medium leaderboards; friends/guild scope |
| Sharded sorted sets, synchronous fan-out | Exact | O(shard count) latency per rank query | Medium-large scale, rank/range queries are core UX |
| Sharded sorted sets, cached aggregate | Slightly stale (seconds) | Cheap reads, background refresh cost | Large scale, Top-N and casual rank checks |
| Percentile/bucket approximation (e.g., t-digest-style histogram) | "You're in the top 5%" — no exact ordinal rank | Very cheap, sublinear in entity count | Extreme scale where exact ordinal rank isn't actually the product requirement |

**Staff-level framing:** the decision isn't "exact vs. approximate" in the abstract — it's *matching the precision the product actually needs to the cheapest structure that delivers it.* A casual mobile game showing "Top 100" and "you're in the top 10%" has no business paying for exact-fan-out global rank on every screen load; a competitive ranked-ladder game where "did I just overtake my rival" is the core loop cannot get away with an approximation.

---

## 18. Senior vs Staff Answer Differentiators

### Senior-level expectations

- Identify the sorted-set data structure (Redis ZSET) as the right tool
- Explain Top-N, rank lookup, and range-around-entity as the three core queries
- Mention caching for read-heavy traffic
- Handle obvious failure modes (node crash, cache miss)
- Provide basic capacity math

### Staff-level expectations (additional)

| Topic | What Staff-level adds |
|---|---|
| Ranking engine | Recognize the single-node scaling wall and design an explicit sharding strategy — not just "use Redis" |
| Sharding strategy | Contrast entity-hash vs. score-range sharding, and explain which read pattern each makes expensive |
| Rank aggregation | Explicit fan-out-and-merge or prefix-sum algorithm, with per-shard timeout and fallback |
| Consistency | Name the exact-vs-approximate rank tradeoff and connect it to PACELC's latency-vs-consistency axis explicitly |
| Time windows | Distinguish fixed windows (trivial, new key) from rolling windows (hard, needs bucket-merge or decay) |
| Tie-breaking | Encode tiebreak into the sort key itself; use server-assigned sequence, not client clock |
| Anti-cheat | Treat client-submitted scores as adversarial input; server-authoritative scoring where feasible |
| Hot keys | Address both write hotspots (score-range clustering) and read hotspots (celebrity entity) as distinct problems with distinct mitigations |
| Real-time push | Push rank deltas to subscribed entities only, not raw score events to everyone |
| Failure modes | Proactively enumerate shard hotspots, fan-out stragglers, cache stampedes, and cheating — not just node crashes |

### The single most important Staff differentiator

**Recognizing that a single Redis sorted set is a textbook toy answer, and that the real design problem begins exactly where that structure stops fitting on one node.** The choice of sharding strategy (entity-hash vs. score-range) is not a minor implementation detail — it determines which of the three core queries (Top-N, rank, range-around-me) becomes cheap and which becomes an expensive fan-out. A Staff-level candidate names this tradeoff explicitly and picks a strategy based on which query the product actually needs to be cheap; a Senior-level candidate stops at "we shard the sorted set."

---

## 19. Interview Time Allocation

For a 45-minute system design session:

| Phase | Time | Focus |
|---|---|---|
| Requirements & scope | 5 min | Confirm query shapes (Top-N, rank, range), scale, freshness needs |
| Capacity estimation | 5 min | Events/sec, read:write ratio, memory footprint, shard count |
| High-level architecture | 5 min | Draw ingestion → Kafka → ranking engine → aggregation → read API |
| Ranking engine & sharding (deep dive) | 12 min | Sorted set primitives, entity-hash vs. score-range tradeoff, rebalancing |
| Rank aggregation & consistency | 8 min | Fan-out/prefix-sum algorithms, exact vs. approximate rank, PACELC framing |
| Time windows & scopes | 5 min | Fixed vs. rolling windows; scope multiplication |
| Tie-breaking & anti-cheat | 3 min | Composite sort key; adversarial input handling |
| Failure modes | 5 min | Hot shards, fan-out stragglers, cache stampedes, cheating |
| Real-time push (if time) | 2 min | Delta-only, subscription-scoped notifications |

**What to cut if short on time:** real-time push detail and anti-cheat depth. **Never cut:** the sharding-strategy tradeoff and the exact-vs-approximate rank discussion — these are the two places the interview is actually differentiating Senior from Staff.

---

## 20. Quick-Reference Cheatsheet

```
KEY DATA STRUCTURE
───────────────────
Sorted Set (Redis ZSET):  skip list + hash table
  ZADD:              O(log N)   insert/update score
  ZREVRANK:           O(log N)   rank lookup
  ZREVRANGE:          O(log N+M) top-N or range-around-entity
  Scaling wall:        single-threaded, single-node — shards past a
                        few tens of thousands of writes/sec sustained

SHARDING STRATEGIES
────────────────────
Entity-hash:    even write distribution  |  global rank = full fan-out + merge
Score-range:    cheap rank via prefix-sum |  hot-shard risk on score clustering
Hybrid:         hash shards for writes + periodically rebuilt range index for reads

TIE-BREAKING
─────────────
composite_score = (raw_score × 10^13) − server_timestamp_ms
  → single ZADD, single sort dimension, earlier achiever ranks higher
  → NEVER use client-supplied timestamps

TIME WINDOWS
─────────────
Fixed (daily/weekly/monthly/all-time):  new key per window — cheap
Rolling (trailing N days):              per-day buckets + read-time merge (exact)
                                          OR decay-weighted score (approximate, cheap)

CAP / PACELC
─────────────
Per-shard ranking:        CP (single-threaded Redis, locally consistent)
Global aggregate rank:    AP, PACELC else-Latency (fan-out cost vs. staleness)
Persistent score store:   AP, tunable consistency (Cassandra/DynamoDB)
Top-N cache:               AP, PACELC else-Latency (stale-but-fast)

HOT-KEY PROBLEMS (two distinct ones)
──────────────────────────────────────
Write hotspot:   score-range bucket absorbs skewed traffic → dynamic rebalance
Read hotspot:    celebrity entity's rank queried excessively → per-entity
                 cache + request coalescing (single-flight)

STAFF-LEVEL SIGNALS TO HIT
───────────────────────────
✓ Name the single-node ZSET scaling wall explicitly
✓ Contrast entity-hash vs. score-range sharding and their query-cost tradeoffs
✓ Fan-out-and-merge or prefix-sum algorithm for cross-shard rank
✓ Exact-vs-approximate rank framed as a PACELC latency/consistency choice
✓ Fixed vs. rolling time windows as genuinely different engineering problems
✓ Tie-break encoded into the sort key with server-assigned sequence
✓ Server-authoritative / adversarial framing for score submission
✓ Two distinct hot-key problems: write (score clustering) vs. read (celebrity)
✓ Kafka as durable log enabling Redis to be a disposable, rebuildable index
✓ Proactive failure mode enumeration (fan-out stragglers, cache stampedes, cheating)
```

---

*Guide compiled for Senior & Staff-level system design interview preparation.*
*Topics: Leaderboard System, Sorted Sets, Sharding, Rank Aggregation, CAP/PACELC, Real-Time Systems.*
*Cross-references: Sharding Strategies · Apache Kafka · CAP/PACELC Theorem · Distributed Unique ID Generation · Distributed Key-Value Store.*
