# Coalescing — System Design Interview Guide
### Senior & Staff Engineer Level

---

## Table of Contents

1. [What Coalescing Is](#1-what-coalescing-is)
2. [Why It Matters: The Thundering Herd](#2-why-it-matters-the-thundering-herd)
3. [Type 1: Request Coalescing (Single-Flight)](#3-type-1-request-coalescing-single-flight)
4. [Type 2: Write Coalescing (Batching)](#4-type-2-write-coalescing-batching)
5. [Type 3: Event / Signal Coalescing](#5-type-3-event--signal-coalescing)
6. [Applying It Across This Repo's System Designs](#6-applying-it-across-this-repos-system-designs)
   - [6.1 Web Crawler](#61-web-crawler)
   - [6.2 URL Shortener](#62-url-shortener)
   - [6.3 Twitter](#63-twitter)
   - [6.4 News Feed System](#64-news-feed-system)
   - [6.5 Leaderboard](#65-leaderboard)
   - [6.6 Messaging App (WhatsApp)](#66-messaging-app-whatsapp)
   - [6.7 Instagram](#67-instagram)
   - [6.8 Netflix](#68-netflix)
   - [6.9 Leader Election](#69-leader-election)
   - [6.10 Office 365 App Usage Alert System](#610-office-365-app-usage-alert-system)
7. [CAP / PACELC Positioning](#7-cap--pacelc-positioning)
8. [Failure Modes & Mitigations](#8-failure-modes--mitigations)
9. [Scalability & Sharding](#9-scalability--sharding)
10. [Senior vs Staff Answer Differentiators](#10-senior-vs-staff-answer-differentiators)
11. [Interview Time Allocation](#11-interview-time-allocation)
12. [Quick-Reference Cheatsheet](#12-quick-reference-cheatsheet)
13. [Related Guides](#13-related-guides)

---

## 1. What Coalescing Is

**Coalescing** means merging many identical or similar operations into one, so the expensive work runs once and every caller shares the result.

It is not one technique. It is a family of three, each solving a different problem:

| Type | Merges | Solves | Classic examples |
|---|---|---|---|
| **Request coalescing** | Concurrent identical *reads* | Cache-miss stampedes | Go `singleflight`, Caffeine `AsyncLoadingCache`, nginx `proxy_cache_lock`, Varnish request collapsing |
| **Write coalescing** | Many small *writes* | Round-trip and syscall overhead | Kafka `batch.size` + `linger.ms`, Redis pipelining, `BF.MADD`, DB group commit |
| **Event coalescing** | Repeated *notifications* for the same key | Unbounded queues from bursty updates | Dirty-flag sets, Kafka log compaction, UI render debouncing |

---

## 2. Why It Matters: The Thundering Herd

Once you put a cache in front of anything slow, you create two new problems:

1. **Read side:** N callers miss the same key at the same moment and all hit the slow backend together.
2. **Write side:** high-rate small writes spend most of their time on network round-trips, not on work.

```
WITHOUT COALESCING                         WITH REQUEST COALESCING

 Caller 1 ──miss──┐                         Caller 1 ──miss──┐ (leader)
 Caller 2 ──miss──┤                         Caller 2 ──wait──┤
 Caller 3 ──miss──┼──▶ 200 × backend        Caller 3 ──wait──┼──▶ 1 × backend
   ...            │    requests               ...            │    request
 Caller N ──miss──┘                         Caller N ──wait──┘
                                                  ▲
                                                  │ all waiters receive
                                                  │ the leader's result
```

**The herd gets worse exactly when you can least afford it:** cold start, cache flush, a popular key expiring, or a backend that is already slow (slow responses widen the window in which more callers pile up).

---

## 3. Type 1: Request Coalescing (Single-Flight)

### How it works

The first caller to miss a key becomes the **leader** and performs the fetch. Every later caller for the same key, while the fetch is in flight, attaches to the leader's future and receives the same result.

```java
ConcurrentHashMap<String, CompletableFuture<RobotsRules>> inFlight = new ConcurrentHashMap<>();

CompletableFuture<RobotsRules> get(String host) {
  RobotsRules cached = cache.get(host);
  if (cached != null) return CompletableFuture.completedFuture(cached);

  return inFlight.computeIfAbsent(host, h ->
      fetchRobots(h)
        .orTimeout(5, TimeUnit.SECONDS)
        .whenComplete((rules, err) -> {
          if (rules != null) cache.put(h, rules);
          inFlight.remove(h);           // always clear, success or failure
        }));
}
```

**Why `computeIfAbsent`:** it is atomic per key, so exactly one future is created per key even under contention. The `remove` in `whenComplete` ensures a failed fetch doesn't stay cached as an in-flight future forever.

### Implementation options

| Option | How | Pros | Cons |
|---|---|---|---|
| **In-process single-flight** | Map of key → in-flight future per node | Zero network cost; trivial; lowest latency | Dedups only *within* a node. 36 nodes means up to 36 fetches per key |
| **Distributed lock** (Redis `SET NX PX`) | Leader takes the lock; others poll or subscribe for the result | Cluster-wide dedup | Extra round-trip on every miss. Lock TTL tuning is hard: too short gives duplicate work, too long stalls waiters if the leader dies |
| **Shard-affinity routing** | Route by `hash(key)` so one node owns each key, then single-flight in-process | Cluster-wide dedup with no locks | Hot shards; dedup depends on routing staying correct during rebalances |
| **Stale-while-revalidate** | Serve the expired entry; one background refresh updates it | Callers never block; smooths TTL-expiry spikes | Only when stale data is acceptable. Doesn't help when there is no entry at all |
| **Probabilistic early refresh** (XFetch) | Each reader refreshes early with a probability that rises near expiry | No coordination at all | Some duplicate refreshes still happen; harder to reason about |

### Choosing between them

```
Is stale data acceptable for this key?
  ├── Yes → stale-while-revalidate for the EXPIRY path
  │         (still need single-flight for the cold-MISS path)
  └── No  → must block on miss
             │
             Is traffic for a key already routed to one node?
               ├── Yes → in-process single-flight (cluster-wide for free)
               └── No  → is a duplicate fetch harmful or just wasteful?
                          ├── Harmful (e.g., hits a rate-limited third party)
                          │     → distributed lock
                          └── Just wasteful
                                → in-process single-flight; accept ≤ N-node duplicates
```

---

## 4. Type 2: Write Coalescing (Batching)

### How it works

Buffer operations and flush them as one batch when **either** a size limit **or** a time limit is reached, whichever comes first.

```
 op ─┐
 op ─┤      ┌───────────────────┐    flush when:
 op ─┼────▶ │  Buffer           │    • size ≥ batch.size   OR
 op ─┤      │  (dedup in place) │    • age  ≥ linger.ms
 op ─┘      └─────────┬─────────┘
                      │ 1 round-trip
                      ▼
                  Backend
```

### Variants

| Technique | Example | Pros | Cons |
|---|---|---|---|
| **Size/time batching** | Kafka `linger.ms`; Redis pipelining; `BF.MADD` | 10–100× throughput; fewer syscalls and round-trips | Adds latency up to the linger time; buffered data lost on crash before flush |
| **Last-write-wins coalescing** | Collapse 10 updates to `last_crawl_ts` for one domain into 1 write | Removes redundant writes entirely | Only valid when intermediate states don't matter |
| **Additive coalescing** | Merge `inbound_links += 1` ×1000 into `+= 1000` | Turns hot-key contention into one write | Needs commutative operations; values are delayed |
| **Dedup-in-buffer** | Drop duplicate URLs inside the batch before calling the Bloom filter | Saves backend work; real inputs are highly redundant | Memory for the buffer's hash set |

### The core tradeoff

> Batching trades **latency** and **crash durability** for **throughput**.

That trade is right when the downstream is tolerant of loss or re-processing (AP stores, idempotent consumers). It is wrong when every write must be durable before acknowledging the caller, unless the batch is flushed synchronously before the ack (as in database **group commit**).

---

## 5. Type 3: Event / Signal Coalescing

### How it works

Instead of queueing every event, keep **only the latest state per key** plus **one pending notification per key**.

```
Events in:   A1  B1  A2  A3  B2  A4
                       │
Naive queue: [A1, B1, A2, A3, B2, A4]   ← grows with burst size

Coalesced:   dirty = {A, B}             ← bounded by number of keys
             state = {A: A4, B: B2}
```

The durable version is **Kafka log compaction**: only the latest record per key survives.

| Pros | Cons |
|---|---|
| Queue size bounded by key count, not event rate | Loses intermediate history |
| Consumer always processes the freshest state | Bad fit for audit, analytics, or anything needing every transition |
| Absorbs bursts naturally | Consumer can't tell how many changes happened |

---

## 6. Applying It Across This Repo's System Designs

Coalescing shows up, in one of the three flavors, in nearly every design in this repo's `HighLevelSystemDesign` library — it's the generic fix for "many identical reads/writes/notifications converge on one hot thing," and almost every large-scale design has at least one hot thing. Below is the full guide's worth of detail for the Web Crawler (6.1), followed by the same mapping, more briefly, for every other design in the library that uses this pattern.

### 6.1 Web Crawler

| Crawler hotspot | Problem | Coalescing type | Recommended approach |
|---|---|---|---|
| **robots.txt cache miss** | New domain → hundreds of fetchers request `/robots.txt` at once, violating the politeness you promised | Request | Domain-shard affinity + in-process single-flight; negative-result caching for 404 |
| **DNS cache miss** | Same burst, against your resolver's rate limit | Request | Per-node single-flight in the local DNS cache; negative caching for NXDOMAIN |
| **DNS / robots.txt expiry** | Popular entries expire together and stampede | Request | Stale-while-revalidate + TTL jitter |
| **Bloom filter checks** | ~11.5K pages/s × ~50 links ≈ 575K checks/s | Write | Dedup-in-buffer, then `BF.MADD` / pipelined batches |
| **Frontier inserts** | Same volume of small writes | Write | Kafka producer batching (`linger.ms` 5–10ms) |
| **`inbound_links` counters** | Hot pages get thousands of increments | Write (additive) | Aggregate counts in memory, flush periodically |
| **`next_ok_ts` updates** | Repeated schedule changes per domain | Event | Dirty-domain set; scheduler reads latest state |
| **New-domain queue creation** | Two fetchers race to create a back queue (Web Crawler guide §6) | Request | Shard affinity makes it local; otherwise `SET NX` |

#### Why shard affinity matters here

The crawler already routes by `hash(domain)` for politeness. That gives you **cluster-wide request coalescing at no extra cost**: all requests for `example.com` land on one node, so an in-process single-flight map dedups across the whole fleet without any distributed lock.

*(Source: [`03_WebCrawler.md`](../../HighLevelSystemDesign/03_WebCrawler.md))*

### 6.2 URL Shortener

| Hotspot | Problem | Coalescing type | Recommended approach |
|---|---|---|---|
| **Hot short-code redirect reads** | A viral link's lookups concentrate on one Redis key/shard | Request | In-process L1 cache ([§11](../../HighLevelSystemDesign/01_url_shortener_system_design.md#11-component-5-caching-layer)); single-flight so an expired hot key causes one DB read instead of a hundred |
| **Click-analytics writes** | 1M clicks/hour on one URL causes write contention and Kafka partition imbalance | Write | Local aggregation (buffer clicks in memory), then repartition by `hash(user_ip + short_code)` instead of `hash(short_code)` alone ([§12 Hot-Key Problem](../../HighLevelSystemDesign/01_url_shortener_system_design.md#12-component-6-analytics--tracking)) |
| **Cache TTL expiry** | Many cached short codes expire together | Request (expiry) | TTL jitter (±5–10%) + stale-while-revalidate; negative caching for 404/410 |

*(Source: [`01_url_shortener_system_design.md`](../../HighLevelSystemDesign/01_url_shortener_system_design.md))*

### 6.3 Twitter

| Hotspot | Problem | Coalescing type | Recommended approach |
|---|---|---|---|
| **Like/retweet/view counters** | Synchronous per-event row/document increments create hot-row contention on viral tweets | Write (additive) | Stream write-coalescing — aggregate counts via the event stream, never a synchronous increment ([§17](../../HighLevelSystemDesign/05_Twitter_system_design.md#17-component-13-counters-likes--retweets--views)) |
| **Viral tweet's content cache key** | One cache node saturates under a hot-partition read pattern | Request | Request coalescing + read-through replica fanout for hot keys ([§19 Failure Modes](../../HighLevelSystemDesign/05_Twitter_system_design.md#19-failure-modes--mitigations)) |
| **Timeline cache eviction storm** | Many users' timeline caches evicted simultaneously under memory pressure, each needing an expensive rebuild | Request (expiry) | Staggered TTLs + per-user rebuild locks so concurrent requests for the same evicted user don't all trigger redundant rebuilds |

*(Source: [`05_Twitter_system_design.md`](../../HighLevelSystemDesign/05_Twitter_system_design.md))*

### 6.4 News Feed System

| Hotspot | Problem | Coalescing type | Recommended approach |
|---|---|---|---|
| **Viral post read stampede** | A celebrity post causes a read stampede on the same `post_id` in the Post Store | Request | Request coalescing — serve concurrent identical reads from one shared in-flight fetch ([§13 Celebrity Problem](../../HighLevelSystemDesign/07_News%20Feed%20System.md#13-the-celebrity-problem-fan-out-asymmetry)) |
| **Celebrity post fan-out** | Pushing a post to millions of followers' feed caches at once would saturate the fan-out tier | Write | Hybrid fan-out: push for normal accounts, pull-time merge for celebrities ([§7](../../HighLevelSystemDesign/07_News%20Feed%20System.md#7-component-3-fan-out-service-push-vs-pull-vs-hybrid)) |

*(Source: [`07_News Feed System.md`](../../HighLevelSystemDesign/07_News%20Feed%20System.md))*

### 6.5 Leaderboard

| Hotspot | Problem | Coalescing type | Recommended approach |
|---|---|---|---|
| **Hot-entity rank reads** | A celebrity entity's rank lookup overwhelms the aggregation service | Request | Per-entity read cache with single-flight — concurrent requests for the same hot entity share one upstream lookup ([§10](../../HighLevelSystemDesign/13_Leaderboard_system_design.md#10-component-6-read-api--caching-layer)) |
| **Top-N cache expiry** | A cache miss under peak load triggers simultaneous full re-aggregation | Request (expiry) | Stale-while-revalidate, or a short single-flight lock so only one request recomputes ([§15 Failure Modes](../../HighLevelSystemDesign/13_Leaderboard_system_design.md#15-failure-modes--mitigations)) |

*(Source: [`13_Leaderboard_system_design.md`](../../HighLevelSystemDesign/13_Leaderboard_system_design.md))*

### 6.6 Messaging App (WhatsApp)

| Hotspot | Problem | Coalescing type | Recommended approach |
|---|---|---|---|
| **Read-receipt fanout in large groups** | Pushing a receipt event per reader in real time would amplify to group-size × message-count | Event | Aggregate read state ("Read by 47") and batch/debounce receipt fanout rather than push per-reader events ([§10](../../HighLevelSystemDesign/08_Messaging_App_like_WhatsApp.md#10-component-5-delivery-state-machine--read-receipts)) |
| **Push notification storm** | Many pending messages for one device each trigger their own push, hitting APNs/FCM rate limits | Write/Event | Coalesce multiple pending messages into one "you have N new messages" push per device ([§15](../../HighLevelSystemDesign/08_Messaging_App_like_WhatsApp.md#15-component-10-push-notification-bridge)) |

*(Source: [`08_Messaging_App_like_WhatsApp.md`](../../HighLevelSystemDesign/08_Messaging_App_like_WhatsApp.md))*

### 6.7 Instagram

| Hotspot | Problem | Coalescing type | Recommended approach |
|---|---|---|---|
| **CDN origin thundering herd on a viral post** | The first wave of viewers all miss-hit the CDN edge simultaneously the instant a celebrity post becomes visible, hammering the origin | Request | Pre-warm CDN edges with popular renditions *before* the post becomes visible, rather than coalescing reactively after the herd starts ([§8](../../HighLevelSystemDesign/10_Instagram_system_design.md#8-component-4-feed-generation--fan-out-strategy)) |
| **Celebrity post fan-out** | Same push-amplification problem as Twitter/News Feed, compounded by media-processing being on the critical path | Write | Hybrid fan-out; fan-out can't even start until media processing completes ([§17 Failure Modes](../../HighLevelSystemDesign/10_Instagram_system_design.md#17-failure-modes--mitigations)) |

*(Source: [`10_Instagram_system_design.md`](../../HighLevelSystemDesign/10_Instagram_system_design.md))*

### 6.8 Netflix

| Hotspot | Problem | Coalescing type | Recommended approach |
|---|---|---|---|
| **Cold EVCache after a deploy** | A fresh/cold cache tier means every request simultaneously misses and hits the backing store (Cassandra) at once | Request | Request coalescing + staggered TTL jitter + warm-up before cutting over traffic — the same thundering-herd signature as any read-heavy service's cache stampede ([§15 Failure Modes](../../HighLevelSystemDesign/11_Netflix_system_design.md)) |

*(Source: [`11_Netflix_system_design.md`](../../HighLevelSystemDesign/11_Netflix_system_design.md))*

### 6.9 Leader Election

| Hotspot | Problem | Coalescing type | Recommended approach |
|---|---|---|---|
| **Leader-change notification** | If every follower watched the leader directly, a leader change would wake all N−1 followers simultaneously, and they'd all re-race to read the child list | Event | Watch-the-predecessor, not watch-the-leader — each node only reacts to its immediate predecessor's change, spreading the notification instead of broadcasting it to everyone at once ([§8](../../HighLevelSystemDesign/15_LeaderElection_system_design.md#8-component-4-external-coordination-service-recipe)) |

This is a structurally different kind of coalescing than the cache-miss cases above: it's not deduping *redundant fetches*, it's reshaping a one-to-N broadcast into a chain so no single coalescing point has to absorb all N wakeups at once.

*(Source: [`15_LeaderElection_system_design.md`](../../HighLevelSystemDesign/15_LeaderElection_system_design.md))*

### 6.10 Office 365 App Usage Alert System

| Hotspot | Problem | Coalescing type | Recommended approach |
|---|---|---|---|
| **Login-hour query spike (e.g. everyone logging in around 9am)** | Synchronized client polling creates a request thundering herd against the query API/cache | Request | Client-side poll jitter + per-user cache + exponential backoff ([§16 Failure Modes](../../HighLevelSystemDesign/14_Office_365_app_usage_alert_system.md#16-failure-modes--mitigations)) |
| **Duplicate usage events** | At-least-once delivery can double-report the same usage event | Write (idempotent) | Idempotent `max()` merge on `last_used_at` makes duplicates harmless by construction — not something to coalesce away, but worth naming as the alternative to a dedup buffer |

*(Source: [`14_Office_365_app_usage_alert_system.md`](../../HighLevelSystemDesign/14_Office_365_app_usage_alert_system.md))*

---

## 7. CAP / PACELC Positioning

| Mechanism | Positioning | Reasoning |
|---|---|---|
| In-process single-flight | Local only; no partition concerns | No distributed state. Under a partition, each side coalesces independently and you get at most one fetch per side |
| Distributed lock (Redis `SET NX`) | **CP-leaning** | Correctness depends on a single lock holder. On partition or Redis failover, fail *open* (allow the duplicate fetch) for caches; fail *closed* only where a duplicate is truly harmful |
| Stale-while-revalidate | **AP** | Explicitly chooses availability (serve stale) over consistency |
| Write batching | **PACELC: EL** | Even without a partition, trades latency for throughput |
| Event coalescing / log compaction | **AP / eventual** | Consumers converge on latest state; intermediate states are dropped by design |

**Rule of thumb:** a coalescing layer in front of a cache should **fail open**. If the coalescing mechanism breaks, you want duplicate work, not an outage.

---

## 8. Failure Modes & Mitigations

| Failure | Symptom | Detection | Mitigation |
|---|---|---|---|
| **Leader failure poisons waiters** | All waiters receive the same error | Error spikes correlated per key | Decide policy: propagate, or let one waiter retry as new leader. Don't cache transient errors |
| **Slow leader stalls everyone** | p99 latency for a key matches the slowest fetch | Per-key wait-time histogram | Leader timeout (`orTimeout`); waiter-side timeout; fall back to stale if available |
| **In-flight entry never cleared** | Key permanently "loading"; callers hang | Count of in-flight entries older than timeout | Clear in `finally` / `whenComplete`; sweeper for orphaned entries |
| **Unbounded waiter list** | Memory growth on a hot key | Waiters-per-key gauge | Cap waiters; shed load or serve stale beyond the cap |
| **Distributed lock holder dies** | Waiters stall for the full lock TTL | Lock age monitoring | Short TTL with leader heartbeat/extension; waiters fall back to fetching after a bounded wait |
| **Synchronized expiry** | Periodic stampedes at TTL boundaries | Backend QPS sawtooth | TTL jitter (±10–20%); stale-while-revalidate; probabilistic early refresh |
| **Batch lost on crash** | Writes in the buffer disappear | Gap between produced and acknowledged counts | Keep linger short; idempotent replay from upstream (Kafka offsets not committed until flush) |
| **Batch latency spike under low load** | Each write waits the full linger time | Latency rises when traffic drops | Flush on size OR time; keep linger small (single-digit ms) |
| **Negative results not coalesced** | Dead domains stampede repeatedly | High NXDOMAIN / 404 rate per key | Cache and coalesce negative results with a short TTL |

---

## 9. Scalability & Sharding

### How coalescing effectiveness scales

```
Duplicate backend calls per key per miss window:

  In-process only, random routing:   ≤ number of nodes (N)
  In-process + shard affinity:       1
  Distributed lock:                  1 (plus lock round-trip on every miss)
```

At 36 fetcher nodes, in-process-only coalescing still allows up to 36 robots.txt fetches for a new domain. Shard affinity drops that to one without adding a network hop.

### Rebalancing

When shards move (node added or removed), two nodes may briefly both own a key:
- **Effect:** at most one duplicate fetch per moving key during the handoff window.
- **Acceptable** for caches (AP); use consistent hashing to limit how many keys move.
- **Not acceptable** where a duplicate is harmful; there you need a lock or fencing token during handoff.

### Hot keys

Coalescing helps hot *misses*, not hot *hits*. A key that is hit 1M times/s with a valid cache entry needs a different fix: replicate the entry to multiple nodes or to a near-cache in each client.

---

## 10. Senior vs Staff Answer Differentiators

### Senior-level expectations

- Know that caches can stampede on misses
- Mention request collapsing or a lock to prevent duplicate fetches
- Use batching to improve write throughput

### Staff-level expectations (additional)

| Topic | What Staff-level adds |
|---|---|
| Taxonomy | Distinguishes request, write, and event coalescing and picks one per hotspot |
| Cluster scope | Notices in-process coalescing only dedups per node; uses existing shard affinity for cluster-wide dedup without locks |
| Expiry vs miss | Separates cold-miss stampedes (single-flight) from expiry stampedes (stale-while-revalidate, TTL jitter) |
| Negative caching | Coalesces and caches failures (NXDOMAIN, 404), not just successes |
| Failure semantics | Leader failure, slow leader, orphaned in-flight entries, waiter caps |
| Fail-open principle | Coalescing breakage should produce duplicate work, never an outage |
| Batching tradeoff | Names latency + crash durability vs throughput, and ties the choice to downstream consistency (AP, idempotent) |
| Integration | Connects coalescing to earlier design decisions (domain sharding, AP frontier) rather than bolting it on |

### The single most important Staff differentiator

**Raising the stampede unprompted.** The moment you put a cache in front of a slow or rate-limited dependency, say: "this cache has a herd problem on miss and on expiry, and here's how I handle each." Waiting for the interviewer to ask is a Senior-level answer.

---

## 11. Interview Time Allocation

Coalescing is rarely its own interview question. It is a **2–4 minute deep dive** inside a larger design. Typical insertion points:

| When | Where in a 45-min session | Time | What to say |
|---|---|---|---|
| You introduce any cache | High-level architecture (min ~15) | 30s | Flag the stampede risk; promise to return to it |
| Deep dive on a cached dependency | Deep dives (min ~20–35) | 2 min | Single-flight + shard affinity; stale-while-revalidate + jitter |
| Throughput math looks tight | Capacity estimation / deep dive | 1 min | Write batching; quote the round-trip savings |
| Failure modes section | Failure modes (min ~38) | 1 min | Leader failure, slow leader, fail-open |

### Sample script

> "Once you have a cache in front of anything slow (DNS, robots.txt, the Bloom filter), you have a stampede problem on misses and a round-trip problem on writes. I'd coalesce reads with single-flight, made cluster-wide for free by the domain-sharded routing we already have. I'd coalesce writes with pipelined batching, using a short linger like 5–10ms since the frontier is AP. Then I'd add stale-while-revalidate and TTL jitter so expirations don't synchronize."

---

## 12. Quick-Reference Cheatsheet

```
THREE TYPES
───────────
Request coalescing:  N concurrent identical reads  → 1 backend call
Write coalescing:    N small writes                → 1 batched call
Event coalescing:    N updates for one key         → latest state + 1 notification

REQUEST COALESCING OPTIONS
──────────────────────────
In-process single-flight     cheapest; per-node dedup only
+ shard affinity             cluster-wide dedup, no locks
Distributed lock (SET NX)    cluster-wide; extra RTT; TTL tuning pain
Stale-while-revalidate       for EXPIRY, not cold miss; needs stale tolerance
Probabilistic early refresh  no coordination; some duplicates

STAMPEDE CHECKLIST
──────────────────
✓ Cold miss       → single-flight
✓ Expiry          → stale-while-revalidate + TTL jitter
✓ Negative result → cache + coalesce failures (short TTL)
✓ Leader timeout  → waiters don't hang on a slow leader
✓ Always clear    → remove in-flight entry on success AND failure
✓ Fail open       → broken coalescing = duplicate work, not outage

BATCHING TRADEOFF
─────────────────
Gain:  throughput (10–100×), fewer round-trips
Pay:   latency (≤ linger), crash durability (buffer loss)
Flush: size OR time, whichever first
Fits:  AP stores, idempotent consumers

CRAWLER MAPPING
───────────────
robots.txt / DNS miss   → shard affinity + single-flight
robots.txt / DNS expiry → stale-while-revalidate + jitter
Bloom filter checks     → dedup-in-buffer + BF.MADD
Frontier inserts        → Kafka linger.ms 5–10ms
Link counters           → additive coalescing
next_ok_ts updates      → dirty-key set
```

---

## 13. Related Guides

See [§6](#6-applying-it-across-this-repos-system-designs) for the full per-design mapping. Direct links:

- **Web Crawler** ([`03_WebCrawler.md`](../../HighLevelSystemDesign/03_WebCrawler.md)): §6 URL Frontier (new-domain race condition), §12 DNS Cache, §13 robots.txt Cache, §16 Scalability & Sharding (domain-hash routing that enables lock-free coalescing)
- **URL Shortener** ([`01_url_shortener_system_design.md`](../../HighLevelSystemDesign/01_url_shortener_system_design.md)): §11 Caching Layer (L1 single-flight), §12 Analytics & Tracking (Hot-Key Problem, local aggregation)
- **Twitter** ([`05_Twitter_system_design.md`](../../HighLevelSystemDesign/05_Twitter_system_design.md)): §17 Counters (write-coalescing via stream aggregation), §19 Failure Modes (hot partitions, eviction storms)
- **News Feed System** ([`07_News Feed System.md`](../../HighLevelSystemDesign/07_News%20Feed%20System.md)): §7 Fan-out Service, §13 Celebrity Problem (read-side request coalescing)
- **Leaderboard** ([`13_Leaderboard_system_design.md`](../../HighLevelSystemDesign/13_Leaderboard_system_design.md)): §10 Read API & Caching Layer, §15 Failure Modes (Top-N cache stampede)
- **Messaging App (WhatsApp)** ([`08_Messaging_App_like_WhatsApp.md`](../../HighLevelSystemDesign/08_Messaging_App_like_WhatsApp.md)): §10 Delivery State Machine & Read Receipts, §15 Push Notification Bridge
- **Instagram** ([`10_Instagram_system_design.md`](../../HighLevelSystemDesign/10_Instagram_system_design.md)): §8 Feed Generation (CDN pre-warm vs. thundering herd)
- **Netflix** ([`11_Netflix_system_design.md`](../../HighLevelSystemDesign/11_Netflix_system_design.md)): §15 Failure Modes (cold-cache stampede after deploy)
- **Leader Election** ([`15_LeaderElection_system_design.md`](../../HighLevelSystemDesign/15_LeaderElection_system_design.md)): §8 External Coordination Service Recipe (watch-the-predecessor pattern)
- **Office 365 App Usage Alert System** ([`14_Office_365_app_usage_alert_system.md`](../../HighLevelSystemDesign/14_Office_365_app_usage_alert_system.md)): §16 Failure Modes (login-hour thundering herd)
- **Caching strategies guide**: [`02_Caching.md`](02_Caching.md) — cache-aside, TTL design, hot-key promotion/demotion
- **Kafka internals guide** (if in your library): producer batching, `linger.ms`, log compaction

---

*Guide compiled for Senior & Staff-level system design interview preparation.*
*Topics: Coalescing, Thundering Herd, Single-Flight, Batching, Log Compaction, Cache Stampede.*
