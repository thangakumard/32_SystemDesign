# System Design: Redis Deep Dive

> Based on the Hello Interview "Redis" deep dive ([hellointerview.com/learn/system-design/deep-dives/redis](https://www.hellointerview.com/learn/system-design/deep-dives/redis)), with cross-references to where these exact patterns already show up in this repo's system designs.

---

## Table of Contents

1. [Why Redis Comes Up So Often](#1-why-redis-comes-up-so-often)
2. [Core Architecture](#2-core-architecture)
   - [2.1 Single-Threaded Execution](#21-single-threaded-execution)
   - [2.2 Persistence: RDB vs AOF](#22-persistence-rdb-vs-aof)
3. [Data Structures](#3-data-structures)
4. [Replication & Clustering](#4-replication--clustering)
   - [4.1 Replication](#41-replication)
   - [4.2 Clustering (Hash Slots)](#42-clustering-hash-slots)
   - [4.3 Eviction Policies](#43-eviction-policies)
5. [Capability Playbook](#5-capability-playbook)
   - [5.1 Caching](#51-caching)
   - [5.2 Distributed Locking](#52-distributed-locking)
   - [5.3 Leaderboards](#53-leaderboards)
   - [5.4 Rate Limiting](#54-rate-limiting)
   - [5.5 Proximity Search](#55-proximity-search)
   - [5.6 Event Sourcing with Streams](#56-event-sourcing-with-streams)
   - [5.7 Pub/Sub Messaging](#57-pubsub-messaging)
6. [The Hot-Key Problem](#6-the-hot-key-problem)
7. [When Not to Use Redis](#7-when-not-to-use-redis)
8. [Redis vs. Alternatives](#8-redis-vs-alternatives)
9. [Interview Checklist](#9-interview-checklist)
10. [Quick Reference Summary](#10-quick-reference-summary)

---

## 1. Why Redis Comes Up So Often

Redis isn't just "a cache" — it's a **versatile in-memory data structure store**. The same server that backs your cache-aside layer can also hold a distributed lock, a leaderboard, a rate limiter, a geospatial index, and an event log. That versatility, plus a dead-simple execution model (single-threaded, atomic commands, microsecond latency), is exactly why it shows up across so many unrelated interview problems in this repo — not because every design "needs a cache," but because Redis's primitives map directly onto common sub-problems (ordering, counting, deduping, coordinating).

The skill being tested isn't "do you know Redis exists" — it's **do you reach for the right data structure for the sub-problem**, and **do you know where Redis's guarantees stop** (durability, cross-key atomicity, multi-node transactions) so you reach for Kafka/ZooKeeper/SQL instead when the problem actually needs them.

---

## 2. Core Architecture

### 2.1 Single-Threaded Execution

- Redis executes commands **sequentially on a single thread** — a deliberate design choice, not a limitation to work around.
- This buys **atomicity for free**: any single command (`INCR`, `ZADD`, a Lua script) runs to completion without interleaving from another client. No locks, no race conditions to reason about for single-key operations.
- In-memory storage + single-threaded execution → **microsecond** command execution on the server, sub-millisecond round trips over the network.
- Throughput: a single node handles **~100,000 writes/second**. That number is worth memorizing — it's the first sanity check for "does this fit on one Redis node" math (the same kind of back-of-envelope check used throughout this repo, e.g. the write-fan-out math in [`Fb_post_search_system_design.md`](../../HighLevelSystemDesign/Fb_post_search_system_design.md)).

### 2.2 Persistence: RDB vs AOF

Redis is fast *because* it trades away durability guarantees by default — know this trade-off cold, because "is this data safe to lose" is almost always the right follow-up question in an interview.

| Mechanism | How it works | Failure mode |
|---|---|---|
| **RDB** (snapshotting) | Periodic point-in-time snapshot of the whole dataset | A crash between snapshots loses everything written since the last one |
| **AOF** (append-only file) | Logs every write command | Default `fsync` is once per second (configurable) — up to ~1s of writes can be lost on crash |

- The trade-off is **intentional**: Redis prioritizes speed over durability. If you need disk-backed durability without giving that up, mention **AWS MemoryDB** as the alternative that makes the opposite trade (durable, but slower).
- Takeaway for interviews: Redis is a fine place for a **derived/rebuildable** copy of data (cache, counters, index), but a risky place for **the single source of truth** — see [§7](#7-when-not-to-use-redis).

---

## 3. Data Structures

Redis keys are strings; values can be any of these structures. Picking the right one *is* the design decision — most Redis-based system design problems reduce to "which structure models this access pattern in O(log n) or better."

| Structure | What it models | Typical interview use |
|---|---|---|
| **String** | Simple value / counter | Cache entries, `INCR`-based counters |
| **Hash** | Object/dictionary | Storing a struct-like record under one key |
| **List** | Ordered sequence | Simple queues |
| **Set** | Unordered unique collection | Membership checks, dedup |
| **Sorted Set (ZSET)** | Ordered-by-score collection, backed by a skip list + hash table | Leaderboards, rate-limit windows, priority queues |
| **Stream** | Append-only log | Event sourcing, work queues with consumer groups |
| **Geospatial Index** | Geohash-encoded coordinates, stored internally as a sorted set | Proximity search |
| **Bloom filter, JSON, Time Series** (Redis 8+) | Specialized structures | Probabilistic membership, document storage, time-series metrics |

---

## 4. Replication & Clustering

### 4.1 Replication

- Replication is **asynchronous**: the primary acknowledges a write to the client *before* replicas have received it.
- **Failure mode to know:** if the primary dies right after acknowledging a write but before replicating it, a promoted replica can be missing that write. This is the concrete mechanism behind "Redis can lose recently-acknowledged writes on failover" — don't just assert it, explain why.
- Replicas can absorb **read traffic** if the client is configured for `READONLY` reads against them — a cheap way to scale reads horizontally without sharding.

### 4.2 Clustering (Hash Slots)

- Redis Cluster splits the keyspace into **16,384 hash slots**. Each key hashes to a slot; each slot is assigned to a node. That assignment *is* the sharding mechanism.
- **There is no query router inside Redis Cluster** — clients must know (or discover) which node owns which slot. Clients cache the slot→node mapping and connect directly; a `MOVED` reply (when a slot has migrated) tells the client to refresh its map.
- **Multi-key operations require all keys to live on the same slot.** This is the single biggest clustering gotcha: `MGET`, transactions, and Lua scripts touching multiple keys all fail across slots unless you force colocation.
- **Hash tags** force colocation: `{user:123}:posts` and `{user:123}:likes` both hash on `user:123` (the part inside `{}`), landing on the same slot even though the full key strings differ. Use hash tags whenever a design needs multiple related keys to be operated on together.

### 4.3 Eviction Policies

- **Default:** Redis rejects new writes once memory is full — correct behavior if Redis is a system of record for something, but surprising if you expected a cache.
- **Cache mode:** set `allkeys-lru` so Redis evicts the least-recently-used key instead of rejecting writes.
- LRU here is **approximate** (sampling-based), not exact — Redis samples a handful of keys and evicts the "most stale" among them rather than tracking a true global LRU order. This is a deliberate cost/accuracy trade-off, and it's fine to say so if asked.

---

## 5. Capability Playbook

This is the part that actually gets tested: given a sub-problem, which Redis primitive solves it, and what's the exact command sequence.

### 5.1 Caching

- The simplest mapping: cache keys = Redis keys, values = cached payload, TTL manages staleness.
- Scales horizontally the same way any Redis use case does — shard by key across cluster nodes.
- For the broader caching patterns (cache-aside, write-through, write-behind, stampede/thundering-herd mitigation, hot-key remediation at the caching-layer level), see [`02_Caching.md`](02_Caching.md) — this document stays focused on what's specific to Redis as the engine behind that cache.

### 5.2 Distributed Locking

Basic lock acquisition is one atomic command:

```
SET lock:concert:343 my-token NX EX 30
```

- `NX` — succeeds only if the key doesn't already exist (so only one client can "win" the lock).
- `EX 30` — a 30-second expiry, so a crashed holder doesn't block the lock forever.
- **Release must be a Lua script** that checks the token before deleting — otherwise client A can delete a lock that actually belongs to client B (because A's lock expired and B re-acquired it in between A's check and A's delete).

**Redlock** (acquiring the lock across a majority of independent Redis nodes) is the extended version for surviving a single primary's failure — but it's controversial. Martin Kleppmann's critique: Redlock reasons about wall-clock TTLs, not a logical quorum-verified epoch, so clock drift or a paused process (GC, VM stall, swap) can let two clients simultaneously believe they hold a valid lock. Fencing tokens mitigate this but aren't built in.

**Staff framing:** treat Redis-based locks as **efficiency tools that occasionally fail**, not correctness guarantees. [`15_LeaderElection_system_design.md`](../../HighLevelSystemDesign/15_LeaderElection_system_design.md) makes this exact distinction explicit — it uses `SET leader_key <node_id> NX PX 10000` for a cheap, ad-hoc lock, but is explicit that it is *not linearizable* and reaches for Raft/ZAB-backed systems (etcd/ZooKeeper) whenever leadership correctness actually matters, for precisely the clock-drift/pause reasons Kleppmann describes.

### 5.3 Leaderboards

Sorted sets give O(log n) insert/update and O(log n + k) ranked range reads — exactly the shape a leaderboard needs:

```
ZADD tiger_posts 500 "SomeId1"              # insert/update score
ZREMRANGEBYRANK tiger_posts 0 -6            # trim to top-5 (negative index counts from the end)
ZREVRANGE tiger_posts 0 99                  # top-100, descending
```

- Writing the **absolute score** (not a delta) via `ZADD` is naturally idempotent — replaying the same write is a no-op, which matters for at-least-once delivery.
- This is the core data structure in [`13_Leaderboard_system_design.md`](../../HighLevelSystemDesign/13_Leaderboard_system_design.md), which also covers the next question an interviewer asks: **what happens when one sorted set no longer fits on one node** (sharded leaderboards + a fan-in merge step, `ZREVRANGE` on every shard in parallel then merged). The milestone-based ZSET updates in [`Fb_post_search_system_design.md`](../../HighLevelSystemDesign/Fb_post_search_system_design.md) (§ "Write path (like)") are the same primitive applied to a likes-ranked postings list instead of a pure leaderboard.

### 5.4 Rate Limiting

| Approach | Mechanism | Trade-off |
|---|---|---|
| **Fixed window** | `INCR` a counter keyed by current window; reject once it exceeds N. Set the expiry **only when `INCR` returns 1** (first request in the window) — otherwise every request resets the TTL and the window never closes. | Simple, but allows bursts at window boundaries |
| **Sliding window** | A sorted set keyed per client, score = request timestamp. Atomically (Lua script): `ZREMRANGEBYSCORE` (drop entries older than the window) → `ZCARD` (count what's left) → `ZADD` (record this request). | Smooths boundary bursts; costs more per request |

The Lua-script atomicity matters here for the same reason as the lock-release script above: without it, two concurrent requests can both read a stale count before either writes, letting both through when only one should pass.

### 5.5 Proximity Search

- `GEOADD` / `GEOSEARCH` give geospatial indexing via **geohashes**, internally stored as a sorted set (geohash integer as the score).
- Search complexity is **O(N + log M)**: N = candidates from the grid cells overlapping the search radius, M = exact matches within that radius after filtering.
- This is the mechanism underlying nearby-search in [`ProximityServices/01_Yelp_system_design.md`](../../HighLevelSystemDesign/ProximityServices/01_Yelp_system_design.md) and [`ProximityServices/20_Local_Delivery_Service.md`](../../HighLevelSystemDesign/ProximityServices/20_Local_Delivery_Service.md) — both reduce to "pick a geo index (geohash vs. quadtree) and justify it" rather than reinventing spatial search from scratch.

### 5.6 Event Sourcing with Streams

- `XADD` appends an entry to a stream; consumer groups coordinate processing via `XREADGROUP`; `XCLAIM`/`XAUTOCLAIM` reassign work from a worker that died mid-processing.
- **Durability caveat:** stream entries are only as durable as Redis's general persistence settings (§2.2) — a stream is not automatically safer than any other Redis-resident data. This is the key fact that separates it from Kafka (see [§8](#8-redis-vs-alternatives)).

### 5.7 Pub/Sub Messaging

- **At-most-once, ephemeral**: no message persistence. A subscriber that's offline when a message is published simply never sees it.
- Channels are created on-demand, no configuration needed; connection overhead is per-node, not per-channel.
- **Sharded Pub/Sub** (`SPUBLISH`/`SSUBSCRIBE`) routes a channel to the node owning its slot, letting pub/sub scale horizontally with the cluster instead of fanning every message through every node.
- **Vs. a homegrown subscriber-tracking scheme:** native pub/sub is preferable — it collapses what would be three network hops (app → tracking service → subscriber) into two, and avoids building/maintaining per-server heartbeat and registry logic yourself.
- **Vs. Streams:** pub/sub has no catch-up story; streams do (a reconnecting consumer can resume from its last-acked ID). Reach for streams, not pub/sub, whenever a consumer needs to replay what it missed.

---

## 6. The Hot-Key Problem

**Scenario:** load isn't evenly distributed across keys — one key (a viral post, a mega-event, a celebrity profile) absorbs disproportionate traffic and overwhelms the single node/slot that owns it. Scaling out the cluster doesn't help, because every request for that key still lands on the same node.

| Remediation | How it works | Cost |
|---|---|---|
| **Client-side (in-process) caching** | App servers keep a small local cache of the hottest items with a short TTL, absorbing requests before they ever reach Redis | Must manage coherence — a short TTL bounds staleness instead of eliminating it |
| **Key copies / key splitting** | Duplicate the data under N suffixed keys (`product:123:1` … `product:123:10`) that hash to different slots; readers pick a suffix at random | Only helps reads; requires detecting which keys are hot enough to need splitting; writes must fan out to all copies |
| **Read replicas** | Multiplies read capacity if clients are configured to read from replicas | Writes still land on the primary only — doesn't help write-hot keys |

This exact pattern — recognize the hot-key risk and design the remediation **up front**, not as an afterthought — is why it recurs across this repo: [`21_Ticketmaster.md`](../../HighLevelSystemDesign/21_Ticketmaster.md) frames its entire queueing problem as "one Taylor-Swift-sized event is a hot-key problem, not a horizontal-sharding problem," and [`Fb_post_search_system_design.md`](../../HighLevelSystemDesign/Fb_post_search_system_design.md) splits hot search terms into sub-shards for the same reason. The in-process L1 cache pattern described in [`02_Caching.md` §2.2](02_Caching.md#22-in-process-caching) is the "client-side caching" row above, applied generally.

---

## 7. When Not to Use Redis

- **You need durability guarantees for a system of record.** Redis's default persistence trades durability for speed (§2.2); don't put the only copy of business-critical data there.
- **The working set can't economically fit in RAM.** Redis's speed comes from being in-memory — once the dataset is large enough that RAM becomes the dominant cost driver, a disk-based store (or a hot/cold tiering strategy, as in [`Fb_post_search_system_design.md`](../../HighLevelSystemDesign/Fb_post_search_system_design.md)) is the right call.
- **Query flexibility is required.** No joins, and in cluster mode, no cross-key queries unless the keys are forced onto the same slot via hash tags (§4.2).
- **You need durable, replayable streams with long retention for many independent consumers.** That's Kafka's job, not Redis Streams' (see §8).

---

## 8. Redis vs. Alternatives

| Need | Reach for Redis when… | Reach for the alternative when… |
|---|---|---|
| **Event log / streaming** | It's a modest queue already living inside a Redis-centric design — background jobs, notifications, simple work distribution | You need long retention, replay for many independent consumer groups, and durable ordered throughput where message loss is unacceptable → **Kafka** |
| **Coordination / leadership** | A cheap, "mostly works" lock is fine and an occasional double-acquisition is tolerable | Correctness genuinely depends on exactly-one-leader (no split brain ever) → **ZooKeeper/etcd** (Raft/ZAB-backed, quorum-verified, not wall-clock-TTL-based) — see [`15_LeaderElection_system_design.md`](../../HighLevelSystemDesign/15_LeaderElection_system_design.md) |
| **Relational queries** | Access pattern is key-based lookups, ranges, or ordered sets | You need joins, ad-hoc queries, or multi-row transactions across arbitrary keys → **SQL** |

---

## 9. Interview Checklist

- [ ] Can you justify *why* Redis fits this sub-problem (which data structure, what complexity) rather than asserting "we'll cache it in Redis"?
- [ ] Do you know Redis's persistence failure modes (RDB snapshot gap, AOF's 1s fsync default) and when that's acceptable vs. disqualifying?
- [ ] Can you explain hash-slot clustering, why multi-key ops need hash tags, and what a `MOVED` reply means?
- [ ] Do you proactively flag the hot-key risk for *this specific* access pattern, and name a concrete remediation (client cache, key splitting, read replicas)?
- [ ] Do you know when to walk away from Redis — durability-critical systems of record, datasets that don't fit in RAM, correctness-critical coordination, long-retention multi-consumer streaming?
- [ ] For locks: do you call out that single/multi-node Redis locks aren't linearizable, and why (clock drift, process pauses) — not just that Redlock is "controversial"?

---

## 10. Quick Reference Summary

| Capability | Primary structure | Key command(s) |
|---|---|---|
| Caching | String | `GET` / `SET` with `EX` |
| Distributed lock | String | `SET key token NX EX ttl` + Lua-verified `DEL` |
| Leaderboard | Sorted Set | `ZADD`, `ZREVRANGE`, `ZREMRANGEBYRANK` |
| Fixed-window rate limit | String | `INCR` + conditional `EXPIRE` |
| Sliding-window rate limit | Sorted Set | `ZREMRANGEBYSCORE` + `ZCARD` + `ZADD` (atomic via Lua) |
| Proximity search | Geospatial (sorted set internally) | `GEOADD`, `GEOSEARCH` |
| Event sourcing | Stream | `XADD`, `XREADGROUP`, `XCLAIM`/`XAUTOCLAIM` |
| Pub/Sub | Channel | `PUBLISH`/`SUBSCRIBE`, `SPUBLISH`/`SSUBSCRIBE` (sharded) |

**Numbers worth memorizing:** ~100k writes/sec on a single node; 16,384 hash slots in a cluster; AOF fsync default of 1s; `O(N + log M)` for geosearch.
