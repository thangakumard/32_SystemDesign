# Distributed Cache — System Design Interview Guide
### Senior & Staff Engineer Level

> All numbers in this guide are **illustrative** interview-grade estimates, not figures published by any vendor (Redis, Memcached, ElastiCache, etc.). Decimal (SI) units throughout.

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
10. [Deep Dive 1: Low Latency — Sub-Millisecond p99](#10-deep-dive-1-low-latency--sub-millisecond-p99)
11. [Deep Dive 2: Scalability — Elastic Sharding & Hot Keys](#11-deep-dive-2-scalability--elastic-sharding--hot-keys)
12. [Deep Dive 3: High Availability — Replication, Failover, Cold Starts](#12-deep-dive-3-high-availability--replication-failover-cold-starts)
13. [Deep Dive 4: Consistency — Bounded Staleness Between Cache and DB](#13-deep-dive-4-consistency--bounded-staleness-between-cache-and-db)
14. [CAP / PACELC Positioning](#14-cap--pacelc-positioning)
15. [Failure Modes & Mitigations](#15-failure-modes--mitigations)
16. [Scalability & Sharding Summary](#16-scalability--sharding-summary)
17. [Senior vs Staff Answer Differentiators](#17-senior-vs-staff-answer-differentiators)
18. [Interview Time Allocation](#18-interview-time-allocation)
19. [Quick-Reference Cheatsheet](#19-quick-reference-cheatsheet)
20. [Related Guides](#20-related-guides)

---

## 1. Problem Statement & Scope

Design a **distributed, in-memory key-value cache** (think Redis Cluster / Memcached fleet / a managed service like ElastiCache) that sits in front of slower systems of record — relational databases, Cassandra, downstream services — and serves hot data in sub-millisecond time to thousands of application servers.

### In scope
- `GET` / `SET` / `DELETE` with TTL, batch reads (`MGET`), atomic ops (`INCR`, compare-and-set)
- Eviction when memory is full; expiration when TTL elapses
- Horizontal sharding, replication, automatic failover, online resharding
- Cache ↔ database consistency patterns (cache-aside, invalidation, leases)
- Multi-tenant namespaces with quotas

### Out of scope (call out, then park)
- Rich data structures in depth (sorted sets, streams) → *Leaderboard System* uses Redis ZSETs
- Durable KV storage as a source of truth → *Distributed Key-Value Store (Dynamo-style)*
- CDN / HTTP edge caching → *Instagram* / *Netflix* delivery sections
- Distributed locking as a correctness primitive → *Leader Election* (and why a cache is the wrong place for it, §14)

### What the interviewer is really testing

| Signal | What it looks like |
|---|---|
| Partitioning | Consistent hashing / hash slots, and *why* `hash(key) % N` causes a database outage on resize |
| Failure thinking | Stampedes, hot keys, cold restarts, and lost invalidations raised **unprompted** |
| Consistency honesty | Knowing the cache-aside race that leaves stale data *forever*, and how leases/versions close it |
| Tail latency | Why fan-out (`MGET` across shards) and single-threaded event loops shape p99, not p50 |
| Operational ownership | Memory headroom, fragmentation, fork/COW spikes, connection explosion |

**Anchor constraint (drives everything downstream):** *The cache is not the source of truth — it is a load shield for the database.* Losing cached data is acceptable; serving **wrong** data beyond a bounded window is not; and every cache failure mode (stampede, mass expiry, cold node, hot key) ultimately manifests as a **database outage**. Design decisions are judged by two questions: *"what does the database see when this breaks?"* and *"how long can a stale value survive?"*

---

## 2. Functional Requirements

| # | Requirement | Notes |
|---|---|---|
| FR1 | `GET key` / `MGET keys…` returns the value or a miss | Batch reads span shards |
| FR2 | `SET key value [TTL] [NX\|XX] [CAS]` stores a value with optional expiry and conditions | Conditional sets enable idempotent fills and optimistic concurrency |
| FR3 | `DELETE key` / invalidation removes stale entries, including when the source DB changes | Explicit (app) and implicit (CDC-driven) |
| FR4 | Entries expire at TTL; when memory is full, entries are **evicted** by policy | LRU / LFU / TinyLFU, per-namespace |
| FR5 | Atomic operations: `INCR`, `DECR`, `GETS`/`CAS`, `TOUCH` | Counters, rate limiters, optimistic updates |
| FR6 | Namespaces (tenants) with memory quota, eviction policy, ACLs | Noisy-neighbor isolation |

FR1–FR4 are walked through in detail in §9.

---

## 3. Non-Functional Requirements

| Requirement | Target | Why it matters |
|---|---|---|
| **Low latency** | Server-side p99 < 1 ms; client-observed p99 < 2 ms, p99.9 < 5 ms (same AZ) | A page render may issue 50–500 cache calls; tail latency compounds |
| **Scalability** | 10M ops/s peak, ~4 TB hot data; add/remove nodes online moving ≈ 1/N of keys | Traffic and working set grow continuously; resizes cannot cause miss storms |
| **High availability** | 99.99% for reads; node failure → failover < 10–15 s; no DB meltdown on node loss | A cache outage converts directly into a DB outage |
| **Consistency (cache ↔ DB)** | Staleness bounded: invalidation p99 < 1 s, hard upper bound = TTL; no permanently-stale entries | Stale price/permission data is a correctness bug, not a perf bug |
| Hit ratio | ≥ 99% for well-behaved read-heavy workloads (workload-dependent) | 1 point of hit ratio can double DB load (§4) |
| Durability | **Not required**; best-effort warm restart from snapshot | Cache is rebuildable — but cold rebuild is expensive |
| Isolation | Per-tenant memory quotas, rate limits, eviction independence | One tenant's scan must not evict another's working set |
| Security | TLS in transit, ACLs per namespace, no secrets in keys | Caches often hold PII, session tokens |

**Read:write ratio:** typically 10:1 to 100:1. We assume **90% reads** for sizing.

---

## 4. Capacity Estimation

```
Assumptions (illustrative):
  Peak client-facing ops          = 10M ops/s cluster-wide   (90% reads, 10% writes/deletes)
  Distinct hot keys               = 5B
  Avg key                         = 60 B
  Avg value                       = 500 B
  Per-entry overhead              = ~70 B   (hash bucket, pointers, expiry, LRU/LFU bits)
  Allocator fragmentation         = ×1.2
  Replication                     = 1 primary + 1 replica per shard

Memory:
  5B × (60 + 500 + 70) B          = 3.15 TB
  × 1.2 fragmentation             ≈ 3.8 TB  ≈ 4 TB primary data
  × 2 (replica)                   ≈ 8 TB total RAM for data

Node sizing (per node):
  128 GB RAM box, ~64 GB usable for data
  (headroom: replication/output buffers, fork copy-on-write during snapshots, OS)

Shard count — take the MAX of three bounds:
  Memory-bound:     4 TB / 64 GB              ≈ 63 shards
  Throughput-bound: 10M ops/s ÷ 100K ops/s     = 100 shards
                    (single event-loop node ≈ 200K ops/s; plan at 50% CPU)
  Hot-key bound:    any single key > ~100K ops/s cannot be fixed by more shards (§11.5)
  ⇒ 100 shards × 2 = 200 nodes, ~40 GB data per shard  → THROUGHPUT-BOUND, not memory-bound

Network:
  10M ops/s × ~600 B              ≈ 6 GB/s ≈ 48 Gbps cluster-wide
  per node                        ≈ 0.5 Gbps  (fine on 25 Gbps NICs)
  BUT one 1 MB "big key" read 50K/s = 50 GB/s = 400 Gbps → impossible on one NIC

Replication stream:
  1M writes/s × 600 B             ≈ 600 MB/s cluster ≈ 6 MB/s per shard

Connections (if every app instance talks directly to every shard):
  20K app instances × 100 shards × 2 conns ≈ 4M connections cluster-wide
  ≈ 40K connections per node  → memory for buffers + epoll pressure → proxy or multiplexing

Database shielding (why hit ratio is THE metric):
  9M reads/s at 99% hit           → 90K DB reads/s
                at 98% hit        → 180K DB reads/s   (1 point = 2× DB load)
                at 95% hit        → 450K DB reads/s   (5× — most DBs fall over)
                at 0% (cold)      → 9M DB reads/s     (instant outage)

Warm restart vs cold refill (one 40 GB shard):
  Load snapshot at ~1 GB/s        ≈ 40 s
  Refill from DB at miss rate     ≈ hours, while the DB absorbs every miss
```

**Key insights**
1. **This cluster is throughput-bound, not memory-bound.** Always compute all three bounds (memory, ops/s, hot key) and take the max — saying this out loud is a Staff signal.
2. **Hit ratio is a cliff, not a slope.** Misses scale as `(1 − hit)`, so 99% → 98% doubles DB traffic. Any event that drops hit ratio (resize, failover, mass expiry) is a DB capacity event.
3. **Big keys are NIC-bound, hot keys are CPU-bound** — neither is solved by adding shards.
4. **Connection count** grows as `clients × shards` — a hidden scaling wall.

---

## 5. Core Entities

```
CacheEntry (in-memory, per shard)
  key            bytes (≤ 250 B–1 KB enforced)
  value          bytes (≤ 1 MB default; larger → chunk or reject)
  expire_at      ms epoch (0 = no TTL)
  cas_version    64-bit, incremented on every mutation (optimistic concurrency)
  flags          client-defined (serialization format, compression)
  evict_meta     LRU clock (24 bits) or LFU log-counter + decay time
  namespace_id   tenant

Namespace (tenant config)
  namespace_id, memory_quota_bytes, eviction_policy (LRU|LFU|TTL-only|noeviction),
  max_value_bytes, ops_rate_limit, default_ttl, acl

Shard / Slot
  slot_id ∈ [0, 16383]   (hash slot = CRC16(key) mod 16384)
  owner_shard_id, state (STABLE | MIGRATING | IMPORTING)

Node
  node_id, shard_id, role (PRIMARY | REPLICA), az, address,
  replication_offset, config_epoch, health

ClusterMap (topology — versioned)
  epoch, slot → shard, shard → {primary, replicas[]}

Lease (per key, short-lived — §13)
  key, lease_token (64-bit), holder, issued_at, ttl (~10 s)

InvalidationEvent (from CDC)
  namespace, key, db_version (e.g., row version / LSN), op (UPSERT|DELETE), ts
```

**Why 16,384 fixed hash slots instead of hashing straight onto nodes?** Slots decouple *key → slot* (fixed forever) from *slot → node* (changes on resize). Resharding moves whole slots, the topology map stays small (16K entries), and clients cache it cheaply. See §11.2 for the alternatives.

---

## 6. API / System Interface

Two interfaces: the **data-plane protocol** used by client libraries (RESP/memcached-style binary, not HTTP — HTTP framing alone would blow the latency budget), and the **control-plane API** for operators.

### Data plane (client library view)

```text
GET    key                               → value | MISS
MGET   k1 k2 … kn                        → [value|MISS …]      # library splits by shard, merges
SET    key value [EX sec|PX ms] [NX|XX]  → OK | NOT_STORED
GETS   key                               → (value, cas_version)
CAS    key value cas_version [EX sec]    → OK | EXISTS (version changed) | NOT_FOUND
DEL    key [key…]                        → count
INCR   key [delta]                       → new_value
TOUCH  key EX sec                        → OK                  # extend TTL without fetching

# Lease-based fill (thundering-herd + stale-set protection, §13)
LGET   key                               → value | MISS(lease_token) | MISS(retry_after_ms)
LSET   key value lease_token [EX sec]    → OK | LEASE_INVALID
```

Typical client wrapper used by applications (cache-aside):

```java
V getOrLoad(String key, Duration ttl, Supplier<V> loader) {
    Result r = cache.lget(key);
    if (r.hit()) return r.value();
    if (r.retryAfter()) { sleep(r.retryAfterMs()); return getOrLoad(key, ttl, loader); } // or serve stale
    V v = loader.get();                                   // read DB
    cache.lset(key, v, r.leaseToken(), jitter(ttl));      // dropped if invalidated meanwhile
    return v;
}
```

### Cluster protocol responses (routing)

```text
-MOVED 3999 10.0.4.17:6379      # slot permanently owned elsewhere → refresh topology
-ASK   3999 10.0.4.22:6379      # slot mid-migration → retry this one request there
-TRYAGAIN                       # multi-key op spans a migrating slot
-OOM / -QUOTA_EXCEEDED          # namespace over quota with noeviction policy
```

### Control plane

```http
POST   /v1/namespaces           { "name":"profiles", "memory_gb":200, "eviction":"lfu", "default_ttl_s":3600 }
PATCH  /v1/namespaces/{id}      { "memory_gb":400 }                # triggers scale-out plan
POST   /v1/clusters/{id}/reshard { "target_shards":120 }           → 202 { "plan_id":… }
GET    /v1/clusters/{id}/topology                                  → { "epoch":…, "slots":[…] }
GET    /v1/namespaces/{id}/hotkeys?top=50
POST   /v1/clusters/{id}/failover { "shard":42 }                   # manual / planned
```

**Design choices to defend**
- **Binary/RESP over TCP, persistent connections, pipelining** — per-request TCP or TLS handshakes cost more than the cache lookup itself.
- **Smart client library** owns routing, retries, timeouts, and the topology cache — so most requests are a single network hop.
- **TTL is mandatory by default** (namespace `default_ttl`) — TTL is the last line of defence against permanently stale data.
- **`NX`, `CAS`, and leases** exist so that concurrent fills are safe; a plain `SET` from a miss path is the source of the classic stale-forever race (§13).

---

## 7. Data Flow

### 7.1 Read (cache-aside, FR1)
```
1. App → client lib: getOrLoad(k)
2. Lib: slot = CRC16(k) mod 16384 → shard → node (from cached ClusterMap)
3. Node: hash-table lookup; check expire_at (lazy expiry) → HIT → return
4. MISS → node issues lease token → app reads DB → LSET(k, v, token, ttl±jitter)
5. Node accepts LSET only if lease still valid (no DELETE arrived meanwhile)
```

### 7.2 Write (FR2)
```
1. App writes DB (source of truth) and commits
2. App DELETEs k (preferred) — or SETs k with new value + version
3. Separately, CDC stream emits an invalidation for k → invalidator DELETEs k (safety net)
4. Primary applies, appends to replication stream → replica applies asynchronously
```

### 7.3 Invalidation (FR3)
```
DB commit → binlog/WAL → CDC (Debezium) → Kafka (partition by key)
         → Invalidator workers → map row → cache keys → DEL (+ invalidates any open lease)
         → commit Kafka offset only after DEL acked
```

### 7.4 Expiry & eviction (FR4)
```
On every access:       expired? → delete + MISS                       (lazy)
Background, 10×/s:     sample 20 keys with TTL; delete expired; repeat if >25% expired (active)
On write when used_memory > namespace quota or maxmemory:
                       sample N candidates → evict lowest-score (LRU age / LFU count) until fits
```

### 7.5 Topology change
```
Operator / autoscaler → Control plane → migration plan (slots → new shard)
→ per slot: MIGRATING/IMPORTING → move keys in batches → flip owner → bump epoch
→ clients get MOVED/ASK → refresh ClusterMap
```

---

## 8. High-Level Design

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                         Application Servers (thousands)                           │
│  ┌────────────────────────────────────────────────────────────────────────────┐   │
│  │ Cache Client Library                                                        │   │
│  │ • L1 near-cache (hot keys, 1–5 s TTL)   • ClusterMap cache (epoch)          │   │
│  │ • slot routing CRC16 mod 16384          • conn pool + pipelining            │   │
│  │ • timeouts, hedging, circuit breaker    • lease-aware getOrLoad             │   │
│  └───────────────────────────┬────────────────────────────────────────────────┘   │
└──────────────────────────────┼────────────────────────────────────────────────────┘
                               │ RESP/binary over TCP (TLS)
          (optional) ┌─────────▼──────────┐    used when client count × shards explodes,
                     │  Proxy tier        │    or for non-smart clients (mcrouter /
                     │  (Envoy/Twemproxy- │    Twemproxy / Envoy Redis filter)
                     │   style), stateless│
                     └─────────┬──────────┘
                               │
┌──────────────────────────────▼────────────────────────────────────────────────────┐
│                     Cache Data Plane — 100 shards × (1 primary + 1 replica)        │
│                                                                                    │
│  ┌── Shard 0 (slots 0–163) ──┐  ┌── Shard 1 (slots 164–327) ─┐      ┌── Shard 99 ──┐│
│  │ Primary   (AZ-a)          │  │ Primary   (AZ-b)           │ ...  │ Primary (AZ-c)││
│  │  hash table · TTL index   │  │                            │      │              ││
│  │  evictor · lease table    │  │                            │      │              ││
│  │      │ async repl stream  │  │      │                     │      │     │        ││
│  │      ▼                    │  │      ▼                     │      │     ▼        ││
│  │ Replica   (AZ-b)          │  │ Replica   (AZ-c)           │      │ Replica(AZ-a)││
│  └───────────────────────────┘  └────────────────────────────┘      └──────────────┘│
│        ▲   cluster bus: gossip heartbeats, failure detection, epoch-based failover  │
└────────┼──────────────────────────────────────────────────────────────┬───────────┘
         │                                                               │ RDB snapshot /
┌────────┴──────────────────────┐                          ┌─────────────▼────────────┐
│ Control Plane                 │                          │ Object Store (S3)         │
│ • Config store (etcd, CP)     │                          │ periodic snapshots for    │
│   ClusterMap epochs, quotas   │                          │ warm restart / new replica│
│ • Orchestrator: reshard plans,│                          └──────────────────────────┘
│   node replacement, upgrades  │
│ • Hot-key / big-key detector  │
└───────────────────────────────┘

┌────────────────┐  binlog/WAL   ┌──────────┐   ┌──────────────────┐   ┌───────────────┐
│ Source-of-truth│──────────────▶│ CDC      │──▶│ Kafka            │──▶│ Invalidator   │──▶ DEL
│ DBs (Postgres, │               │(Debezium)│   │ invalidations    │   │ workers       │   to cache
│ Cassandra …)   │◀── miss reads ┘          │   │ partition by key │   │ row → keys    │
└────────────────┘   from app servers       └───┴──────────────────┘   └───────────────┘

┌────────────────────────────────────────────────────────────────────────────────────┐
│ Observability: hit ratio per namespace, p99/p99.9 per node, evictions/s, memory    │
│ fragmentation, replication lag, hot keys, DB miss QPS (the number that matters)    │
└────────────────────────────────────────────────────────────────────────────────────┘
```

### Component responsibilities & why

| Component | Responsibility | Why this choice |
|---|---|---|
| Client library | Routing, pooling, pipelining, retries, L1, leases, breaker | One network hop on the hot path; no extra proxy latency |
| Proxy tier (optional) | Connection multiplexing, routing for dumb clients, request coalescing | Collapses `clients × shards` connections; ~100–300 µs extra hop |
| Cache node | In-memory hash table, TTL, eviction, leases, replication | Single-threaded event loop per shard (or shard-per-core) → no locks |
| Replica | Hot standby; optional read scaling | Failover target with a warm working set |
| Cluster bus (gossip) | Heartbeats, failure detection, failover votes | No external dependency on the data path |
| Config store (etcd) | Authoritative topology epochs, namespace quotas | Topology must be **CP** — two owners for one slot is split brain |
| Orchestrator | Reshard plans, rolling upgrades, node replacement | Automates the operations that cause outages when done by hand |
| Snapshot store | RDB-style snapshots to S3 | Warm restart in seconds instead of hours of DB refill |
| CDC → Kafka → Invalidator | DB-driven invalidation independent of app code | Catches writes from batch jobs, other services, manual fixes (see *Kafka* guide) |

### Functional requirement coverage

| FR | Path through the diagram |
|---|---|
| FR1 GET/MGET | App → client lib (L1?) → shard primary (or replica) → value / miss → DB → LSET |
| FR2 SET | App → client lib → shard primary → replication stream → replica |
| FR3 DELETE / invalidation | App → DEL; DB → CDC → Kafka → Invalidator → DEL |
| FR4 TTL / eviction | Inside each node: lazy + active expiry; sampled eviction under memory pressure |
| FR5 atomic ops | Single shard, single-threaded execution ⇒ atomic by construction |
| FR6 namespaces | Control plane quota → node-level accounting & per-namespace eviction |

---

## 9. HLD Walkthrough: 4 Functional Requirements in Detail

### 9.1 FR1 — GET / MGET

```
App ── getOrLoad("user:42:profile") ──▶ Client lib
  1. L1 near-cache? (only for keys flagged hot) → hit → return (~1 µs)
  2. slot = CRC16("user:42:profile") mod 16384 = 3999 → shard 24 → primary 10.0.4.17
     (hash tags: "user:{42}:profile" hashes only "42" → co-locate a user's keys for MGET)
  3. Pipelined request on pooled connection
Node (event loop):
  4. Lookup in hash table (O(1)); if expire_at < now → delete, treat as MISS
  5. HIT → update LRU clock / LFU counter → write reply
  6. MISS → create lease (token T, 10 s) → reply MISS(T)
Client lib:
  7. On MISS(T): loader() reads DB → LSET(key, value, T, ttl ± 10% jitter)
  8. On -MOVED: refresh ClusterMap (single-flight), retry once
```

**MGET across shards** — group keys by shard, issue one pipelined batch per shard **in parallel**, merge. Latency = **slowest shard** (tail-at-scale: with 20 shards touched and a 1% chance each of a 10 ms stall, ~18% of MGETs see the stall). Controls: hash tags to co-locate keys that are read together, cap fan-out, hedge the slowest shard to its replica.

| Decision | Options | Choice |
|---|---|---|
| Where routing happens | Smart client / proxy / server redirect only | **Smart client + server `MOVED` as correction**; proxy only if connection count or client heterogeneity demands it |
| Reads from replicas? | Primary only / replicas for reads | **Primary by default** (read-your-writes after DEL); per-namespace opt-in replica reads for staleness-tolerant data, **AZ-local** to avoid cross-AZ latency/cost |
| Miss handling | Plain GET→DB→SET / lease-based | **Lease-based** — closes the stale-set race and coalesces stampedes (§13) |

### 9.2 FR2 — SET with TTL and conditions

```
App ── SET k v EX 3600 ──▶ Client lib → primary(shard)
Node:
  1. Namespace quota check: used + size(k,v) > quota? → evict (FR4) or reject (noeviction)
  2. Allocate (jemalloc size class / slab class); insert or replace in hash table
  3. expire_at = now + ttl;  cas_version++
  4. If NX/XX/CAS condition fails → NOT_STORED / EXISTS
  5. Append command to replication backlog (ring buffer) → reply OK
  6. Replica pulls stream asynchronously (offset-based; partial resync from backlog on reconnect)
```

**Value-size policy:** reject or chunk values > 1 MB; compress values > ~1 KB client-side (LZ4/zstd) when CPU is cheaper than bytes. Big values block the event loop during copy and saturate NICs (§4).

**Write strategy options (how SET relates to the DB):**

| Pattern | How | Pros | Cons | Use when |
|---|---|---|---|---|
| **Cache-aside (lazy load)** ✅ default | App reads cache; on miss reads DB, fills cache; on write updates DB then **deletes** key | Simple; cache holds only what's read; cache outage ≠ data loss | Miss latency; stale-set race (needs leases/versions) | General purpose |
| Read-through | Cache itself loads from DB on miss | App code simpler; coalescing built in | Cache must know DB schemas/credentials; couples tiers | Managed loaders (e.g., Caffeine in-process, DAX) |
| Write-through | Write goes to cache, which synchronously writes DB | Cache always fresh for written keys | Write latency = both; caches data never read; cache in the write path's availability | Read-after-write-heavy data |
| Write-behind (write-back) | Write to cache, async flush to DB | Fastest writes; batching | **Data loss** on cache node failure; ordering complexity | Counters/metrics where loss is tolerable |
| Write-around | Write to DB only; cache filled on read | Avoids polluting cache with write-once data | First read always misses | Write-heavy, read-rarely data |
| Refresh-ahead | Refresh hot keys before TTL expiry | No miss latency on hot keys | Wasted refreshes on cold keys | Very hot, predictable keys |

**Why delete-on-write rather than set-on-write?** Two concurrent writers doing "update DB, then SET cache" can apply in opposite orders at DB vs cache → cache permanently holds the loser's value. DELETE is idempotent and order-insensitive; the next read loads the committed value.

### 9.3 FR3 — DELETE and DB-driven invalidation

```
Path A — application-driven:
  App: BEGIN; UPDATE users SET … WHERE id=42; COMMIT;
       DEL user:{42}:profile                         ← after commit, never before
Path B — CDC-driven (safety net and the only path for writes outside the app):
  Postgres WAL → Debezium → Kafka topic "invalidations" (key = table:pk)
     → Invalidator: row → affected cache keys (key-template registry per namespace)
     → DEL keys (also voids any open lease on those keys)
     → commit offset only after DEL acked  (at-least-once; DEL is idempotent)
```

| Decision | Options | Choice |
|---|---|---|
| Who invalidates | App only / CDC only / both | **Both** — app DEL gives fast read-your-writes; CDC catches batch jobs, other services, manual SQL, and app crashes between commit and DEL |
| Delete before or after DB commit | Before / after / both ("double delete") | **After**, plus CDC as a delayed second delete. Delete-before lets a reader refill the old value before commit |
| Row → key mapping | Hard-coded / key-template registry | **Registry** (e.g., `users.id → user:{id}:profile, user:{id}:settings`) — derived/aggregate keys are the usual leak source |
| Ordering | Any order / per-key order | **Kafka partitioned by key** → per-key order; idempotent DEL makes duplicates harmless |
| Multi-region | Per-region invalidators consuming a replicated topic | Each region's cache invalidated from its own consumer group |

**Tag-based / group invalidation** (e.g., "drop everything for tenant 7"): either maintain a set of member keys per tag (costly) or use **namespace versioning** — prefix keys with `ns_version` and bump it; old keys become unreachable and age out via TTL/eviction. Generational invalidation is O(1).

### 9.4 FR4 — Expiration and eviction

**Expiration (correctness: never return an expired value):**

| Approach | Pros | Cons |
|---|---|---|
| Lazy only (check on access) | Zero background cost | Expired-but-never-read keys leak memory |
| Timer per key / sorted TTL index | Precise | Memory + CPU per key; heap ops at millions of keys |
| Hierarchical timing wheel | O(1) schedule/expire | Extra structure; granularity trade-off |
| **Lazy + adaptive active sampling** ✅ (Redis-style) | Cheap; bounds memory held by expired keys | Expired keys may linger briefly (never served) |

**Eviction (memory full):**

| Policy | Pros | Cons | Fits |
|---|---|---|---|
| Exact LRU (hash + doubly linked list) | Simple, recency-aware | 2 pointers/entry (~16 B) + list mutation on every read; **one scan flushes the hot set** | Small in-process caches |
| **Approximated LRU (sample 5–10, evict oldest)** | No list; ~24-bit clock per entry; near-LRU quality | Probabilistic | Default general purpose |
| **Approximated LFU (log counter + decay)** ✅ for skewed workloads | Scan-resistant; keeps truly popular keys | Needs decay to forget yesterday's popularity | Skewed (Zipfian) web traffic |
| W-TinyLFU (Caffeine) | Best hit ratio on many traces; admission filter (count-min sketch) rejects one-hit wonders | More complex; mostly in-process | L1 near-cache |
| Segmented LRU (memcached) | Protects re-referenced items; per-slab-class | Slab calcification (memory stuck in wrong size class) | Fixed-size value workloads |
| FIFO / random | Trivial, lock-free | Lower hit ratio | Extreme throughput, flat access |
| volatile-* (evict only keys with TTL) | Protects "pinned" keys | OOM if non-TTL keys fill memory | Mixed cache + config data |

**Choice:** per-namespace policy; default **approximated LFU** for user-facing data (scan-resistant), LRU for session-like data, W-TinyLFU for the client L1.

**Memory management (where bytes actually go):**
- **jemalloc size classes** (Redis-style) vs **slab allocator** (memcached): slabs give predictable O(1) allocation but suffer *slab calcification* when value sizes shift; jemalloc fragments over time → monitor `fragmentation_ratio` and use **active defragmentation**.
- Account eviction against `maxmemory` **minus headroom** for replication buffers, client output buffers, and fork copy-on-write — the classic cause of the OOM killer taking down a "60%-full" node.

---

## 10. Deep Dive 1: Low Latency — Sub-Millisecond p99

### 10.1 Where latency comes from

```
Client-observed GET (same AZ), illustrative:
  Client lib: hash + pick connection            ~1–2 µs
  Serialization                                 ~1–5 µs
  Network RTT (same AZ)                         ~100–300 µs   ← dominates
  Kernel + event loop dispatch                  ~5–20 µs
  Hash lookup + reply                           ~1 µs
  Queueing behind other commands                0 … many ms   ← dominates the TAIL
Cross-AZ adds ~0.5–1 ms; cross-region adds tens of ms (never on the hot path).
```

The p50 is a **network** problem; the p99 is a **queueing** problem.

### 10.2 Server threading model

| Option | Pros | Cons |
|---|---|---|
| **Single-threaded event loop per node** (Redis classic) | No locks; atomic ops for free; simple | One CPU core per process; one slow command stalls everyone |
| Event loop + I/O threads (Redis 6+) | Parallel socket read/write and parsing; command execution still serial | Helps network-bound workloads; doesn't fix slow commands |
| Multi-threaded with locks (memcached) | Uses all cores in one process | Lock contention on hot buckets / LRU lists |
| **Shared-nothing, shard-per-core** (Dragonfly/Seastar style) ✅ at large box sizes | Linear core scaling, no locks; one big box replaces many processes | Cross-core multi-key ops need message passing; newer ecosystem |

**Choice:** run **multiple single-threaded shard processes per box** (or a shard-per-core engine) — keeps the lock-free model and uses every core. Plan per-shard CPU at ≤ 50%: queueing delay explodes as utilization → 100% (M/M/1: wait ∝ ρ/(1−ρ)).

### 10.3 Eliminate event-loop stalls (the tail killers)

| Stall source | Fix |
|---|---|
| O(N) commands (`KEYS *`, `HGETALL` on huge hash, `SMEMBERS`) | Ban/rename in prod; use cursor-based `SCAN`; cap collection sizes |
| Deleting a 500 MB key | `UNLINK` → background thread frees memory |
| Large value copy (multi-MB) | Max value size; chunking; compression |
| `fork()` for snapshot — page-table copy on a 64 GB process can take 10s–100s of ms; COW copies pages | Snapshot **from replicas only**; huge pages **off** (COW copies 2 MB pages); keep per-process memory moderate (~25–40 GB) |
| Synchronous persistence (`fsync always`) | Not needed for a cache; AOF off or `everysec` on replicas only |
| Mass expiry at the same second | TTL **jitter** (±10%) |
| GC pauses (if built on JVM) | Off-heap storage or a non-GC language (C/C++/Rust) |

### 10.4 Client-side techniques

| Technique | Effect |
|---|---|
| Persistent pooled connections; `TCP_NODELAY` | No handshakes, no Nagle delay |
| **Pipelining** (many requests in flight per connection) | Amortizes RTT and syscalls — 5–10× throughput |
| Batch (`MGET`) + **hash tags** to co-locate keys read together | One round trip, one shard instead of 20 |
| **AZ-affinity** — read replica in client's AZ for staleness-tolerant namespaces | Removes 0.5–1 ms cross-AZ hop and cross-AZ transfer cost |
| **Tight timeouts** (e.g., 5–20 ms) + fall back to DB or stale L1 | A slow cache must not be slower than no cache |
| **Hedged requests** to replica after p95 | Cuts p99.9 for read-only, idempotent ops |
| **L1 near-cache** (in-process, W-TinyLFU, 1–5 s TTL) for hot keys | ~1 µs, zero network; bounded staleness |
| Efficient serialization (protobuf/msgpack; avoid Java serialization) | µs of CPU per call, multiplied by hundreds of calls per page |

### 10.5 Proxy or no proxy?

| Option | Latency | Pros | Cons |
|---|---|---|---|
| Smart client direct | 1 hop | Lowest latency | Needs a mature library per language; `clients × shards` connections |
| Sidecar proxy (per host) | +~50–100 µs (localhost) | Language-agnostic; local multiplexing | Per-host resource cost |
| Central proxy tier | +~100–300 µs (extra network hop) | Connection collapse; central coalescing/routing | Extra tier to scale and fail over |

**Choice:** smart client by default; a sidecar or proxy tier when connection count crosses ~10K+/node or polyglot clients make library maturity a risk.

### 10.6 Latency budget (illustrative, same AZ)

```
                         p50        p99
L1 hit                    1 µs       5 µs
Remote GET hit          200 µs     800 µs   (≤50% CPU per shard, no O(N) commands)
MGET 20 keys, 1 shard   250 µs       1 ms   (hash-tagged)
MGET 20 keys, 10 shards 300 µs     2–3 ms   (tail = slowest shard; hedge helps)
Miss + DB read + LSET   2–10 ms    30+ ms   (why hit ratio is a latency NFR too)
```

---

## 11. Deep Dive 2: Scalability — Elastic Sharding & Hot Keys

### 11.1 The resize problem

With `node = hash(key) % N`, going from 100 → 101 nodes remaps ~**99%** of keys. Every remapped key is a miss → hit ratio collapses to ~1% → DB receives ~9M reads/s → outage. **Partitioning is a DB-protection decision, not just a load-balancing one.**

### 11.2 Partitioning options

| Option | Keys moved on +1 node | Pros | Cons |
|---|---|---|---|
| Modulo `hash % N` | ~all | Trivial | Resize = cache wipe |
| Consistent hashing ring (no vnodes) | ~1/N | Minimal movement | Uneven load; a failed node dumps all load on one neighbor |
| **Consistent hashing + virtual nodes** (100–200 vnodes/node) | ~1/N, spread across all nodes | Even load; weighted nodes; failure load spread | Larger ring metadata; lookups O(log V) |
| Rendezvous (HRW) hashing | ~1/N | No ring; even; simple weighting | O(N) per lookup (fine for ≤ hundreds of nodes) |
| Jump consistent hash | ~1/N | Tiny, fast, perfectly even | Nodes must be numbered 0..N−1; can't remove an arbitrary middle node |
| **Fixed hash slots (16,384) + slot map** ✅ (Redis Cluster) | Exactly the slots you choose to move | Explicit, controllable migration; small map; per-slot state machine | Need a migration protocol; slot count caps max shards |

**Choice: fixed hash slots.** It turns resharding into an explicit, rate-limited plan ("move slots 0–163 from shard 3 to shard 100") instead of an implicit ring perturbation, and lets the orchestrator move hot slots specifically.

### 11.3 Online slot migration (no miss storm)

```
For each slot S moving A → B:
  1. B: mark S IMPORTING from A;  A: mark S MIGRATING to B
  2. Orchestrator loops: keys = A.GETKEYSINSLOT(S, 100); A.MIGRATE(keys → B) (atomic per key)
  3. During migration, requests to A for S:
        key still on A → serve on A
        key already moved → reply -ASK B  (client sends ASKING + cmd to B, one-off)
        multi-key op spanning moved/unmoved keys → -TRYAGAIN
  4. When S is empty on A: set owner(S)=B on A and B, bump config epoch, gossip
  5. Clients hitting A get -MOVED → refresh ClusterMap
Throttle: limit concurrent slots and bytes/s so p99 and replication stay healthy.
```

Data **moves with** the slot — the hit ratio stays intact through the resize. That's the property consistent hashing alone (without data migration) does *not* give you: on a plain ring, the new node starts empty and its 1/N of keys become misses.

### 11.4 Autoscaling signals

| Signal | Action |
|---|---|
| Memory > 70% of usable, or eviction rate rising with hit ratio falling | Scale out (memory-bound) |
| CPU > 50–60% sustained per shard | Scale out or split hot slots (throughput-bound) |
| One slot dominates a shard | Move that slot alone to a lightly loaded shard |
| Connections > threshold | Introduce/scale proxy tier |

Scale **before** peak (predictive, e.g., ahead of sales events) — migration itself consumes CPU and bandwidth.

### 11.5 Hot keys (the domain's "spider trap")

A single key (a celebrity profile, a flash-sale item, a global config) can receive 1M+ reads/s. All of it lands on **one** shard regardless of shard count — this is the same extreme-degree-entity pattern as the crawler's hot domain and Instagram's celebrity.

**Detection:** client libraries sample and report top-K keys (count-min sketch / Space-Saving); servers expose LFU counters; per-slot QPS dashboards; alert when one key > X% of a shard's ops.

| Mitigation | How | Pros | Cons |
|---|---|---|---|
| **L1 near-cache** for flagged hot keys ✅ | In-process, 1–5 s TTL; hot-key list pushed from control plane | Removes ~all remote load; µs latency | Staleness up to L1 TTL; memory per app instance |
| **Key replication / salting** ✅ | Write `k#0 … k#N-1` to different slots; reads pick `k#rand(N)` | Spreads reads across N shards | Writes/invalidations fan out to N copies |
| Read from replicas | Route hot-key reads to all replicas | No app change | Bounded by replica count; replication lag |
| Request coalescing at proxy | Merge identical concurrent GETs into one backend call | Big win for bursty fan-in | Needs proxy tier |
| Dedicated shard | Move hot slot to an isolated, larger node | Isolates blast radius | Still one node's ceiling |

**Hot writes** (e.g., a global counter at 500K INCR/s): split into N sub-counters (`counter#i`), sum on read, or aggregate in-process and flush deltas every 100 ms.

**Big keys** (multi-MB values or million-element collections): enforce max sizes; chunk values (`k:part:0..n`); split collections by sub-key hash; delete with `UNLINK`.

### 11.6 Multi-tenancy at scale

| Option | Pros | Cons |
|---|---|---|
| Shared cluster, key prefixes only | Max utilization | Noisy neighbors; one tenant's scan evicts another's hot set |
| **Shared cluster + per-namespace memory accounting, eviction, rate limits** ✅ | Good utilization with isolation | Accounting overhead per entry |
| Dedicated cluster per tenant (large tenants) | Hard isolation, independent scaling | Fragmented capacity; more ops overhead |

Typical: shared clusters for the long tail, dedicated clusters for the top tenants.

---

## 12. Deep Dive 3: High Availability — Replication, Failover, Cold Starts

### 12.1 What "available" means for a cache

Two distinct failures:
1. **Cache unavailable** → requests fail or time out → apps must fall back to DB → DB overload.
2. **Cache available but cold** (after restart/failover/resize) → hit ratio collapses → DB overload.

Both end at the database. HA here means **keeping the working set warm and reachable**, not preserving data durably.

### 12.2 Replication options

| Option | Pros | Cons |
|---|---|---|
| No replication (memcached-style, client-side hashing) | Simplest; max memory efficiency | Node loss = 1/N of keys cold instantly; DB absorbs the hole |
| **Async primary → replica** ✅ | No write latency penalty; warm standby | Acked writes (incl. DELETEs!) can be lost on failover |
| Semi-sync (`WAIT 1` / min-replicas-to-write) | Bounds loss for critical writes | Write latency + availability tied to replica health |
| Chain replication | Strong consistency, read from tail | Latency = chain length; reconfiguration complexity |
| Quorum (Dynamo-style W+R>N) | Tunable, leaderless | Read repair/hinted handoff overhead — overkill for a cache (see *Dynamo KV* guide) |

**Choice:** async primary-replica with replicas in a **different AZ**; semi-sync only for namespaces that opt in.

### 12.3 Failure detection and failover

```
Gossip bus: every node pings a random subset every 100 ms; tracks last-pong per peer
  1. Node X misses pongs for node_timeout (e.g., 5 s) → marks P "PFAIL" (suspect)
  2. Gossip spreads PFAIL; when a MAJORITY of primaries agree → "FAIL" (confirmed)
  3. Replicas of P wait a rank-based delay (replica with the highest replication offset goes first)
  4. Replica requests votes; primaries grant ≤1 vote per epoch → majority → promotion
  5. New primary takes config_epoch = max+1; announces slot ownership; clients get MOVED
Total: ~node_timeout + election ≈ 5–15 s
```

| Choice | Options | Recommendation |
|---|---|---|
| Who decides | Gossip quorum (Redis Cluster) / external sentinels / control plane via etcd | **Gossip + epochs** for the data path (no external dependency), **etcd** as the authoritative record for the orchestrator. Either way it's a consensus decision — see *Leader Election* |
| Timeout | Short (1–2 s) / longer (5–15 s) | Moderate: too short → false failovers during GC/network blips (each one costs a partial cold start and lost writes) |

### 12.4 Split brain and the stale-DELETE trap (Staff-level correctness)

**Split brain:** the old primary is partitioned from the majority but still reachable by some clients; a replica is promoted on the other side. Both accept writes.
- **Fencing via epochs:** every slot-ownership claim carries `config_epoch`; higher epoch wins when the partition heals; the old primary demotes itself and discards its divergent writes.
- **Self-fencing:** a primary that can't reach a majority of the cluster for `node_timeout` stops accepting writes (`min-replicas-to-write` style). Trade availability for a bounded divergence window.

**The stale-DELETE trap:** async replication can lose an *acknowledged* write. For data, that's fine — it's a cache. But for an **invalidation** it's dangerous:

```
t0  DB: price 10 → 12; app DEL "item:7"   → primary acks; DEL not yet replicated
t1  Primary dies; replica (still holding price=10) promoted
t2  Every reader gets price=10 … until TTL — the DEL is gone forever
```

Mitigations, layered:
1. **TTL as the hard upper bound** on staleness — why TTL is mandatory.
2. **Replay invalidations after failover:** the invalidator rewinds its Kafka consumer group to a timestamp ≈ failover_time − (max replication lag + margin) **for the affected slots** and re-issues DELs (idempotent). This is why invalidations go through a *durable, replayable log* rather than fire-and-forget.
3. **Versioned values:** store `db_version` with the value; readers or the invalidator can detect `cached_version < known_version`.
4. Optional `WAIT 1` on DELETE for high-value namespaces.

### 12.5 Cold-start protection

| Technique | How | Effect |
|---|---|---|
| Warm replicas | Failover target already holds the working set | Failover ≈ no hit-ratio drop |
| **Snapshot-based warm restart** | Periodic RDB snapshot (from replica) to S3; new/restarted node loads it, then catches up via replication | 40 GB in ~40 s vs hours of DB refill |
| **Gutter pool** (Facebook memcache pattern) | Small spare pool absorbs a failed shard's traffic with short TTLs instead of rehashing onto healthy nodes | Prevents cascading overload of neighbors; bounded staleness |
| Gradual traffic ramp | New node joins with low weight; increase as hit ratio climbs | Spreads DB refill over minutes |
| **DB-side admission control** | Client-side circuit breaker + concurrency limit on miss loads; serve stale or degrade feature when breaker opens | The DB survives even if the cache doesn't (see *Circuit Breaker* guide) |
| Request coalescing / leases | One loader per key during refill | Refill load = distinct keys, not requests |

### 12.6 Planned operations without outages
- **Rolling upgrades:** upgrade replicas → planned failover (`FAILOVER` with zero lag wait) → upgrade old primary. Zero data loss, sub-second write pause.
- **Node replacement:** add replica from snapshot, sync, fail over, retire old node.
- **AZ outage:** primaries and replicas spread so each shard has a copy in ≥ 2 AZs; capacity sized for N−1 AZs (each surviving node may get +50% load in a 3-AZ layout).

---

## 13. Deep Dive 4: Consistency — Bounded Staleness Between Cache and DB

### 13.1 Define the guarantee precisely

A cache cannot be linearizable with the DB without making every write a distributed transaction. The achievable, explicit contract:
- **Bounded staleness:** a cached value is at most *Δ* stale after a DB commit — invalidation p99 < 1 s, hard bound = TTL.
- **No permanent staleness:** every stale entry is eventually removed (invalidated or expired).
- **Read-your-writes** for the writing client (delete after commit, then read through to DB).
- Namespaces that need **strong** reads (balances, permissions, inventory at checkout) should **bypass** the cache or validate versions — say this explicitly.

### 13.2 The race that leaves a value stale forever (raise it unprompted)

```
Cache-aside with plain GET → DB → SET:

 Reader R                      Writer W                       Cache
 GET k → MISS
 read DB → v1 (old)
                               UPDATE DB → v2; COMMIT
                               DEL k                          (nothing to delete)
 SET k = v1  ────────────────────────────────────────────────▶ k = v1   ✗ stale until TTL
```

R's fill raced W's invalidation. With long TTLs (or no TTL), this persists for hours. The CDC invalidation can arrive before R's SET too, so CDC alone doesn't fix it.

### 13.3 Options to close it

| Option | How | Pros | Cons |
|---|---|---|---|
| Short TTL only | Staleness bounded by TTL | Trivial | Low hit ratio; still stale for the TTL |
| Delayed double delete | DEL, then DEL again after ~1 s | Simple; catches most races | Heuristic — a slow reader (> delay) still wins |
| **Leases** ✅ (Facebook memcache) | Miss returns a token; any DEL on the key invalidates the token; `LSET` with a voided token is dropped | Closes the race exactly; also rate-limits fills (stampede control) | Server-side lease table; clients must use LGET/LSET |
| **Versioned set (CAS on DB version)** ✅ | Store `(value, db_version)`; SET only if `new_version > cached_version` (server-side compare or Lua) | Monotonic: older data can never overwrite newer | Needs a version column/LSN in the source; tombstones on delete to remember the version |
| Write-through with DB version | Writer updates cache with the committed version | Fresh for written keys | Concurrent writers need version compare anyway |
| CDC "set" instead of "delete" | Invalidator writes the new row image | No miss after writes | Must handle out-of-order events → needs versions; caches unread data |
| Distributed transaction (2PC across DB + cache) | Atomic | — | Wrong tool: latency, availability coupling, cache isn't a resource manager |

**Choice:** **leases for fills + delete-on-write + CDC invalidation + mandatory TTL**, adding **versioned sets** for namespaces where values can be written by multiple paths.

**Lease mechanics:**
```
LGET k   (miss)  → if no lease: issue token T (10 s), reply MISS(T)
                   if lease held by someone else and issued < 2 s ago: reply RETRY(50 ms)  ← stampede control
                   (optionally return the last stale value with a "stale" flag)
DEL k            → remove value AND void lease on k
LSET k v T       → accept only if T is the current, non-voided token
```
In the race above, W's `DEL` voids R's token → R's `LSET v1` is rejected → the next reader loads v2.

### 13.4 Thundering herd / stampede

When a hot key expires or is invalidated, thousands of concurrent requests miss simultaneously and all hit the DB for the same row.

| Technique | How | Trade-off |
|---|---|---|
| **Leases / single-flight** ✅ | Only the lease holder loads; others wait briefly or get stale | Waiters add latency |
| Per-process single-flight (in client) | Coalesce concurrent misses within one app instance | Still N loads for N instances |
| **Stale-while-revalidate** ✅ | Keep a soft TTL (serve) and hard TTL (delete); after soft TTL one request refreshes while others get stale | Bounded extra staleness |
| Probabilistic early expiration (XFetch) | Each reader refreshes early with probability rising as expiry nears (weighted by recompute cost) | No coordination; occasional duplicate refresh |
| TTL jitter | ±10% on TTL | Prevents synchronized mass expiry of keys written together |
| Refresh-ahead for known hot keys | Background job refreshes before expiry | Only for a curated set |

### 13.5 Negative caching and penetration

- **Cache penetration:** requests for keys that don't exist in the DB (bugs or attacks) always miss and always hit the DB. Fix: cache a **tombstone** (`NOT_FOUND`, short TTL, e.g., 30–60 s), and/or a **Bloom filter** of existing IDs in front of the DB (false positive → one harmless DB read; no false negatives — same asymmetry as the *Web Crawler* URL dedup).
- Invalidation must clear tombstones too (a newly created row must not stay "not found").

### 13.6 Consistency across replicas and regions

| Scenario | Guarantee & mechanism |
|---|---|
| Read from primary after own DEL | Read-your-writes (primary applied DEL before ack) |
| Read from replica | Stale by replication lag (ms typically) → only for staleness-tolerant namespaces |
| L1 near-cache | Stale up to L1 TTL (1–5 s); optional pub/sub "evict k" broadcast for faster convergence |
| Multi-region | Each region has its own cache; DB replication + per-region CDC invalidators; cross-region staleness ≈ DB replication lag + invalidation lag. Remote-region reads after a write in the home region may need a "recently written" marker to route to the home DB |
| After failover | Lost DELETEs replayed from Kafka; TTL backstop (§12.4) |

---

## 14. CAP / PACELC Positioning

| Component | Choice | Reasoning |
|---|---|---|
| Cache data plane (values) | **AP / EL** | Serving a slightly stale or missing value beats failing; replicas async; latency over consistency |
| Replica reads / L1 near-cache | **AP / EL** | Explicitly staleness-tolerant by namespace contract |
| Invalidation pipeline (CDC → Kafka) | **AP, at-least-once, per-key ordered** | Duplicates harmless (idempotent DEL); loss is not → durable, replayable log (`acks=all` + `min.insync.replicas` — see *Kafka* guide) |
| **Topology / slot ownership (config epochs, etcd)** | **CP / EC** | Two primaries owning a slot = split brain; consensus required (see *Leader Election*) |
| **Leases & versioned (CAS) sets** | **CP per key** (single primary serializes) | The fill-vs-invalidate decision must be serialized per key; on partition, the self-fenced primary rejects writes |
| Snapshots / persistence | Best effort | Warm-restart optimization only |

Most of the cache is AP/EL; the **named CP exceptions** are topology ownership and per-key lease/version decisions.

> **Staff caution — locks in a cache:** "Redis lock" (`SET NX PX`) or Redlock is fine for **efficiency** (avoid duplicate work), never for **correctness** (mutual exclusion that protects data). Async replication and failover can grant the same lock twice; GC pauses can outlive the lock TTL. For correctness, use a consensus system and **fencing tokens** checked by the resource. Raising this unprompted is a strong Staff signal.

---

## 15. Failure Modes & Mitigations

| Failure | Symptom | Detection | Mitigation |
|---|---|---|---|
| Thundering herd on hot key expiry | DB spike for one row; latency spike | DB QPS per query; miss bursts | Leases/single-flight; stale-while-revalidate; XFetch; TTL jitter |
| Mass synchronized expiry | Periodic DB spikes (e.g., at :00) | Miss-rate seasonality | TTL jitter; refresh-ahead |
| Resize with modulo hashing | Hit ratio → ~0; DB outage | Hit ratio cliff | Hash slots + data migration; throttled reshard |
| Node failure without replica | 1/N keys cold; neighbors overloaded if rehashed | Hit ratio drop by slot | Replicas; gutter pool; DB admission control |
| Failover drops acked DELETEs | Stale values until TTL | Version mismatch audits; post-failover check | Kafka replay of invalidations; TTL; versioned values; `WAIT` for critical ns |
| Split brain | Divergent values; clients on both sides | Two nodes claiming one slot | Config epochs; self-fencing w/o majority; old primary discards writes on rejoin |
| Cache-aside fill race | Value stale for full TTL | Staleness audits (sample cached vs DB) | Leases; versioned sets; delete-after-commit; CDC |
| Lost invalidation (app crash after commit) | Stale until TTL | CDC lag & reconciliation | CDC invalidator as safety net; TTL |
| Hot key | One shard CPU-pegged; p99 up for all keys on that shard | Per-key/top-K telemetry | L1 near-cache; key salting; replica reads; coalescing |
| Big key | NIC saturation; event loop stalls on copy/delete | Big-key scanner; value-size histograms | Size limits; chunking; compression; `UNLINK` |
| O(N) command in prod | Multi-second stall of a shard | Slow log | Rename/disable `KEYS`; cursor `SCAN` |
| Fork / COW during snapshot | Latency spike; memory doubling → OOM | Fork time metric; RSS | Snapshot from replicas; THP off; memory headroom |
| Memory fragmentation | RSS ≫ used memory; early evictions/OOM | `fragmentation_ratio` | Active defrag; jemalloc; restart via planned failover |
| Connection storm (app redeploy) | Accept queue overflow; CPU on TLS handshakes | New-conn/s | Connection pooling; jittered reconnect; proxy multiplexing |
| Cache penetration (non-existent keys) | DB load from always-missing keys | Miss ratio on NOT_FOUND | Tombstones; Bloom filter |
| Noisy neighbor tenant | Another namespace's hit ratio drops | Per-namespace evictions | Per-namespace quotas, eviction, rate limits; dedicated clusters |
| Slow/unavailable cache | App threads blocked → app outage | Client timeout rate | Tight timeouts; circuit breaker; fall back to DB with concurrency limit |
| Cross-AZ reads | Latency + transfer cost | AZ-tagged metrics | AZ-affinity routing |
| Serialization/version mismatch on deploy | Deserialization errors after deploy | Error rate post-deploy | Schema version in key or flags; dual-read during rollout |

**Pattern class:** the hot key and the thundering herd are this domain's version of the crawler's spider trap and Instagram's celebrity fan-out — one entity whose request rate is orders of magnitude above the mean. More shards don't help; it needs a **separate code path** (L1, salting, coalescing).

---

## 16. Scalability & Sharding Summary

| Concern | Approach | What becomes expensive |
|---|---|---|
| Key placement | CRC16 → 16,384 slots → shard map | Multi-key ops across slots → hash tags or client-side fan-out |
| Resize | Throttled slot migration with ASK/MOVED | Migration CPU/bandwidth; must run off-peak or pre-emptively |
| Read scaling | AZ-local replica reads (opt-in) | Staleness; replica capacity |
| Hot keys | L1 + salting + coalescing | Write/invalidation fan-out to N copies |
| Connections | Pooling, pipelining, proxy/sidecar | Extra hop latency with proxies |
| Tenants | Per-namespace accounting; big tenants on dedicated clusters | Capacity fragmentation |
| Multi-region | Independent regional clusters + regional CDC invalidators | Cross-region staleness window |

**Horizontal scaling plan**

| Tier | Lever |
|---|---|
| Cache shards | Add shards, migrate slots; split hot slots |
| Replicas | Add read replicas per shard for read-heavy namespaces |
| Proxy tier | Stateless; autoscale on connections/QPS |
| Invalidators | Scale on Kafka consumer lag (partitions bound parallelism) |
| Control plane | Small; etcd 3–5 nodes |

**The sharding lesson:** key-hash slots make single-key ops one hop and resizes controllable, but turn **batch reads into fan-out** and leave **hot keys** unsolved — the Staff answer names both and handles them explicitly.

---

## 17. Senior vs Staff Answer Differentiators

### Senior-level expectations
- In-memory hash table + TTL + LRU eviction
- Consistent hashing to shard; replicas for HA
- Cache-aside pattern; delete on write
- Mentions thundering herd and hot keys when asked
- Basic memory capacity math

### Staff-level expectations (additional)

| Topic | What Staff adds |
|---|---|
| Anchor | "Cache is a DB load shield" — every failure judged by DB impact and staleness window |
| Capacity | Max of memory-, throughput-, and hot-key bounds; hit-ratio cliff math; connection explosion |
| Partitioning | Why modulo is an outage; slots vs rings; **data moves with the slot** to preserve hit ratio |
| Consistency | Cache-aside fill race raised unprompted; leases / versioned sets; precise bounded-staleness contract; namespaces that must bypass cache |
| Replication | Async replication loses acked DELETEs → Kafka replay + TTL backstop |
| Failover | Epoch fencing, self-fencing, timeout trade-off (false failovers cost cold starts) |
| Cold start | Snapshot warm restart, gutter pool, gradual ramp, DB admission control |
| Tail latency | Event-loop stalls (O(N), fork/COW, big keys), MGET fan-out tail, ≤50% CPU, hedging, AZ affinity |
| Hot keys | Pattern-class framing; L1 + salting + coalescing; hot writes via sub-counters |
| Locks | Cache locks for efficiency only; correctness needs consensus + fencing tokens |
| CAP | AP/EL data plane with named CP exceptions (topology ownership, per-key lease/version) |

**The single most important Staff move:** treating the cache's failure modes as **database capacity events** — and showing a mechanism for each (slot migration, warm replicas, leases, coalescing, admission control) that keeps the DB's miss traffic bounded when the cache misbehaves.

---

## 18. Interview Time Allocation

For a 45-minute session:

| Phase | Time | Focus |
|---|---|---|
| Requirements & scope | 4 min | FR1–FR6; latency, hit ratio, bounded staleness, no durability |
| Capacity estimation | 5 min | 3 bounds → 100 shards; hit-ratio cliff; connections |
| Entities & APIs | 4 min | Slots, ClusterMap epochs, leases, CAS; MOVED/ASK |
| High-level design | 6 min | Client lib, shards+replicas, control plane, CDC invalidation; walk FR1–FR4 |
| Scalability (DD2) | 7 min | Partitioning options, slot migration, hot keys |
| Consistency (DD4) | 7 min | Fill race, leases/versions, stampede, penetration |
| Availability (DD3) | 6 min | Failover, split brain, lost DELETE, cold start |
| Latency (DD1), CAP, failures | 6 min | Event-loop stalls, tail fan-out, CP exceptions, lock caution |

**Cut if short:** allocator details, multi-tenancy options, proxy comparison. **Never cut:** modulo-vs-slots resize argument, the fill race + leases, stampede, and lost-DELETE-on-failover.

---

## 19. Quick-Reference Cheatsheet

```
KEY NUMBERS (illustrative)
──────────────────────────
10M ops/s peak, 90% reads; 5B keys × ~630 B ≈ 3.15 TB → ~4 TB w/ fragmentation
Shards = max(memory 63, throughput 100, hot-key ∞) → 100 shards × 2 = 200 nodes
Per shard: ≤100K ops/s (≤50% CPU), ~40 GB data on a 128 GB box
Network ≈ 48 Gbps cluster; big key 1 MB × 50K/s = 400 Gbps (impossible)
Hit ratio 99% → 90K DB reads/s; 98% → 180K; 95% → 450K
Warm restart 40 GB ≈ 40 s vs hours of cold refill

PARTITIONING
────────────
CRC16(key) mod 16384 → slot → shard; hash tags {…} co-locate keys
Modulo resize moves ~all keys = DB outage; slots move only chosen slots + their data
Migration: MIGRATING/IMPORTING, -ASK during move, -MOVED after; throttle

LATENCY
───────
p50 = network; p99 = queueing → ≤50% CPU, no O(N) cmds, UNLINK, snapshot on replicas
Pipelining, pooled conns, TCP_NODELAY, AZ affinity, tight timeouts, hedging
MGET across k shards → tail = slowest shard
L1 near-cache (W-TinyLFU, 1–5 s) for hot keys

AVAILABILITY
────────────
Async primary→replica (other AZ); gossip PFAIL→FAIL, epoch-voted promotion ~5–15 s
Fencing by config_epoch; primary self-fences without majority
Lost acked DEL on failover → replay invalidations from Kafka; TTL is the backstop
Cold start: snapshot restore, gutter pool, ramp, DB circuit breaker + concurrency cap

CONSISTENCY
───────────
Cache-aside + delete-after-commit + CDC invalidation + mandatory TTL (with jitter)
Fill race → leases (DEL voids token) or versioned CAS sets (only newer wins)
Stampede → leases/single-flight, stale-while-revalidate, XFetch, jitter
Penetration → tombstones + Bloom filter
Strong-read data (balances, permissions) bypasses the cache

EVICTION
────────
Lazy + active-sampled expiry; approximated LRU/LFU by sampling; W-TinyLFU for L1
Headroom for repl buffers, client buffers, fork COW; watch fragmentation

CAP
───
AP/EL: values, replica reads, L1, invalidation log (at-least-once, per-key order)
CP:    slot ownership/topology epochs, per-key lease/version decisions
Cache locks = efficiency only; correctness → consensus + fencing tokens

STAFF SIGNALS
─────────────
✓ Cache is a DB load shield; failures = DB capacity events
✓ Three sizing bounds; hit-ratio cliff
✓ Modulo resize = outage; data moves with slots
✓ Fill race + leases/versions
✓ Lost DELETE after async failover → replay + TTL
✓ Hot key as pattern class (L1 + salting + coalescing)
✓ Event-loop stall sources (O(N), fork/COW, big keys)
✓ Per-component CAP with named CP exceptions; no correctness locks in cache
```

---

## 20. Related Guides

| Guide | Relationship |
|---|---|
| *Distributed Key-Value Store (Dynamo-style)* | Durable sibling: quorum replication, hinted handoff, vnodes — contrast with a non-durable cache |
| *Sharding Strategies* | Consistent hashing, slots, rendezvous; hot-partition bucketing |
| *CAP / PACELC Theorem* | Per-component positioning; replication quorum vs consensus quorum |
| *Leader Election in Distributed Systems* | Epoch-based failover, fencing tokens, why cache locks aren't correctness locks |
| *Apache Kafka* | CDC invalidation log; `acks` + `min.insync.replicas`; replay after failover |
| *Circuit Breaker Pattern* | Protecting the DB when the cache is slow, down, or cold |
| *Instagram* | Hydration caches, timeline cache rebuild stampede, Redis ZSET precision trap |
| *Tinder* | Precomputed stack cache with explicit invalidation; per-user Bloom filters |
| *News Feed System* / *Twitter / Fan-out Architecture* | Timeline caches; celebrity hot keys |
| *URL Shortener* | Canonical read-heavy cache-aside workload; hot short links |
| *Leaderboard System* | Redis sorted sets as a data structure server |
| *Web Crawler* | Bloom filter FP asymmetry (reused for penetration); spider trap ↔ hot key pattern class |
| Behavioral companion | Cache stampede / connection pool exhaustion STAR story — maps to §12.5, §13.4, §15 |

---

*Guide compiled for Senior & Staff-level system design interview preparation.*
*Topics: Distributed Cache, Hash Slots, Consistent Hashing, Leases, Cache Stampede, Hot Keys, Replication & Failover, Cache-DB Consistency.*
