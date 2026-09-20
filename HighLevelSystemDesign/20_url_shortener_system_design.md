# URL Shortener — System Design Interview Guide
### Senior & Staff Engineer Level

---

## Table of Contents

1. [Problem Statement & Scope](#1-problem-statement--scope)
2. [Requirements](#2-requirements)
3. [Capacity Estimation](#3-capacity-estimation)
4. [High-Level Architecture](#4-high-level-architecture)
5. [Component 1: Unique ID Generation](#5-component-1-unique-id-generation)
6. [Component 2: Write Path (Shorten)](#6-component-2-write-path-shorten)
7. [Component 3: Custom URL Aliases](#7-component-3-custom-url-aliases)
8. [Component 4: Read Path (Redirect)](#8-component-4-read-path-redirect)
9. [Component 5: Caching Layer](#9-component-5-caching-layer)
10. [Component 6: Analytics & Tracking](#10-component-6-analytics--tracking)
11. [Component 7: Storage Layer](#11-component-7-storage-layer)
12. [CAP Theorem Positioning](#12-cap-theorem-positioning)
13. [Failure Modes & Mitigations](#13-failure-modes--mitigations)
14. [Concurrency & Collision Handling](#14-concurrency--collision-handling)
15. [Global Distribution & Geo-Routing](#15-global-distribution--geo-routing)
16. [Senior vs Staff Answer Differentiators](#16-senior-vs-staff-answer-differentiators)
17. [Interview Time Allocation](#17-interview-time-allocation)
18. [Quick-Reference Cheatsheet](#18-quick-reference-cheatsheet)

---

## 1. Problem Statement & Scope

A **URL shortener** (think bit.ly, TinyURL, or t.co) is a service that converts long URLs into short, shareable aliases. Users submit a long URL and receive a short code in return; when someone visits the short code, the service redirects them to the original URL.

### What the interviewer is really testing

- Can you design a **high-throughput, low-latency system** that handles both writes and (predominantly) reads?
- Do you understand **collision probability** and how it scales with key space size?
- Can you navigate **tradeoffs between statelessness and state efficiency** (base conversion vs. random generation)?
- Do you proactively address **the hot-key problem** (one link goes viral; all traffic concentrates on a single database row)?
- Can you reason about **multi-master ID generation** at scale without creating collisions?
- Do you understand **CAP/PACELC implications** for a global redirect service?

---

## 2. Requirements

### Functional Requirements

| Requirement | Notes |
|---|---|
| Shorten URLs | Create a short code for a long URL; return it to user |
| Redirect to original | User visits short code; system redirects to original URL |
| Custom short codes | Allow users to specify a vanity short code (e.g., `bit.ly/obama`) |
| URL expiration | Optionally expire short URLs after a time period |
| Editing | Allow users to change the target URL of an existing short code |
| Track analytics | Count clicks, geo, referrer, device type |
| Disable/revoke | Allow users to disable a short URL without deleting the record |

### Non-Functional Requirements

| Requirement | Target |
|---|---|
| Write throughput | 100K new short URLs/day |
| Read throughput | 100M redirects/day (~1,150 redirects/sec) |
| Read latency (P99) | < 100ms |
| Availability | 99.99% (4 nines) |
| Short code length | 6–7 characters (62^6 ≈ 56B unique codes) |
| Scalability | Support growth to 1B+ total shortened URLs |
| Data retention | 10+ years for aged URLs |

---

## 3. Capacity Estimation

```
Write volume:
  100K URLs/day → ~1.16 URLs/sec
  
  Per URL storage:
    Original URL:    ~2KB (average)
    Short code:      6 bytes
    Metadata:        ~500 bytes (created_at, user_id, expires_at, etc.)
    Total per entry: ~2.5 KB
    
  Annual storage:
    100K URLs/day × 365 days × 2.5KB ≈ 91 GB/year
    10 years: ~910 GB total  →  fits in a single DB with heavy caching

Read volume:
  100M redirects/day → ~1,150 redirects/sec
  Peak (10× average): ~11,500 redirects/sec
  
  Connection pool at 100ms avg latency (cache miss):
    11,500 × 0.1s ≈ 1,150 concurrent connections
    With connection pooling (50 connections per app instance): 23 instances
    With caching (cache hit rate 95%): 1-2 instances sufficient

Collision probability (important for Staff-level discussion):
  Key space: 62^6 ≈ 56 billion possible codes
  For 1B URLs, collision probability by birthday paradox:
    P(collision) ≈ (1B)² / (2 × 56B) ≈ 0.009 ≈ 0.9%
    → Expected collisions: ~10M across 1B URLs — manageable but non-negligible

Caching analysis:
  Assuming 95% of reads come from 10% of URLs (power law):
    100M reads/day × 95% = 95M to ~100K URLs
    Average reads per hot URL: 950 reads/day
    Hot cache size: 100K × 2.5KB ≈ 250 MB → fits in Redis
    Miss rate: 5% → 5M cache misses/day → 57 DB queries/sec (easily handled)

Cache hit rate optimization:
  • User clicks on their own shortened URL repeatedly (high locality)
  • Viral links dominate (Pareto distribution)
  • Time-based decay: recently created URLs have higher click rate
```

**Key insight:** The read path is throughput-heavy but the hot set is small. Caching is critical. The write path is trivial in volume but must handle uniqueness guarantees.

---

## 4. High-Level Architecture

```
                    ┌──────────────────────┐
                    │  User / Client       │
                    │  (Web, Mobile, API)  │
                    └──────────┬───────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
    ┌─────────▼────────┐  ┌────▼──────────┐  ┌─▼─────────────┐
    │ Write API        │  │ Read API      │  │ Analytics API │
    │ /shorten         │  │ /redirect/{id}│  │ /stats/{id}   │
    │ (100K/day)       │  │ (100M/day)    │  │ (tracking)    │
    └────────┬─────────┘  └────┬──────────┘  └────┬──────────┘
             │                 │                   │
        ┌────▼─────────────────▼─────────────────┬─┘
        │  API Gateway / Load Balancer           │
        │  Rate limiting, request dedup          │
        └────┬──────────────────────────────────┘
             │
        ┌────┴────────────────────────────────────────┐
        │                                             │
  ┌─────▼──────────────────┐          ┌──────────────▼────────┐
  │ Unique ID Generator    │          │ Cache Layer (Redis)   │
  │ Snowflake / ZK-backed  │          │ Hot URLs + analytics  │
  │ (collision avoidance)  │          │ ~250MB, TTL 24h       │
  └─────┬──────────────────┘          └──────────────┬────────┘
        │                                            │
        │  short_code                                │ miss
        │  (6-7 chars)                               │
        │                                            │
  ┌─────▼──────────────────────────────────────────▼──────────┐
  │ Primary Data Store                                         │
  │ (MySQL / PostgreSQL with strong consistency)               │
  │  Tables: urls, custom_aliases, url_metadata               │
  │  Partitioned by short_code hash for locality               │
  └─────┬───────────────────────────────────────────────────┬─┘
        │                                                   │
        │ Read replicas                                    │ Analytics
        │ (geographic proximity)                           │ stream
        │                                                   │
  ┌─────▼──────────────┐  ┌─────────────────────────────┐  │
  │ Regional Read      │  │ Message Queue (Kafka)       │──┘
  │ Replicas           │  │ Click events                │
  │ (APAC, EU, AMER)   │  │ (geo, referrer, device)     │
  └────────────────────┘  └────────────┬────────────────┘
                                       │
                            ┌──────────▼─────────────┐
                            │ Analytics Pipeline     │
                            │ (Spark/Flink)          │
                            │ → ClickHouse / BigQuery│
                            └────────────────────────┘
```

**Kafka** decouples writes from analytics processing. Analytics can lag without impacting the critical path.

---

## 5. Component 1: Unique ID Generation

The core challenge: generate 6–7 character codes that are:
- **Globally unique** (no collisions across all instances)
- **Non-sequential** (don't leak usage patterns)
- **Collision-free** (must verify availability at write time)
- **Fast** (sub-ms generation on the write path)

### Approach 1: Distributed Snowflake-Style ID (Senior-level)

```
Snowflake structure (64-bit):
  [unused(1) | timestamp(41) | datacenter_id(5) | worker_id(5) | sequence(12)]

Properties:
  • Globally unique across all workers
  • Sortable by time
  • Contains no semantic meaning (good for privacy)
  • 12-bit sequence counter prevents collisions if multiple IDs generated in same ms

Generation process:
  1. Get current timestamp (milliseconds since epoch)
  2. If same ms as last generated: increment sequence counter
  3. If different ms: reset sequence counter
  4. Combine: [timestamp | datacenter | worker | sequence]
  5. Convert 64-bit integer to base-62 string
     (0-9, a-z, A-Z = 62 chars)
     64-bit → 11 base-62 characters → truncate to 6-7 chars
```

**Problem:** Snowflake outputs 11 chars in base-62 for a 64-bit ID. Truncating to 6 chars means collisions are **guaranteed**.

**Why this breaks:** Two consecutive snowflakes, when truncated to 6 chars, have high collision probability.

### Approach 2: Hash-Based Generation (more common in practice)

```
Algorithm:
  1. Generate a candidate short code via:
     a) Random base-62 string (6 chars)  OR
     b) Hash(user_id + timestamp + random_salt) → base-62
  2. Check if code exists in database (via lookup or cache)
  3. If exists → retry with different candidate (exponential backoff)
  4. If doesn't exist → insert and return

Collision resolution:
  Random retry:  O(1) expected tries if collision rate is low
  Hashable:      Deterministic; can embed metadata (user intent)
  
Downsides:
  • Requires database lookup before insertion (critical path overhead)
  • Race condition: two threads may generate same code simultaneously
  • Retry storms possible under high write concurrency
```

### Approach 3: Counter-Based with Base Conversion (Staff-level insight)

Allocate users **a counter range**:

```
Database state:
  next_available_code_id: 1000000000  (a large number, pre-allocated)
  
On shorten request:
  1. Atomic increment: next_available_code_id → fetch-and-add
  2. Convert integer to base-62: 1000000000 → "izq0uq" (6 chars)
  3. Write URL entry keyed on this code
  4. Return code

Properties:
  • No collision checking needed (IDs are sequential)
  • O(1) generation (atomic increment)
  • Fully deterministic
  • Reveals information (sequential pattern)
  • Collision probability: zero
  
Downsides:
  • Counter is a potential bottleneck if single-mastered
  • Predictable codes (user can guess next codes)
  • Reveals usage volume (id growth rate)
```

### Staff-Level Decision Matrix

| Approach | Collision | Speed | Predictable | Best For |
|---|---|---|---|---|
| Snowflake + Base-62 | High (~1–5%) | Fast (sub-ms) | No | Short-term, low-volume |
| Random + Retry | Low (depends on retries) | Slower (DB lookup) | No | Production with 99.9% availability |
| Counter + Base-62 | Zero | Fastest | Yes | High-volume, internal-only services |
| Hybrid: Counter + Shuffle | Zero | Fast | No | Best of both; see below |

### Recommended Approach: Hybrid Counter + Shuffle

```
Setup:
  • Central counter (ZooKeeper, PostgreSQL sequence) pre-allocates batches
  • Each worker gets range: [1B, 1B+1M]
  • Worker converts IDs to base-62 and shuffles locally
  
Pseudocode:
  class ShortCodeGenerator:
    def __init__(self):
      self.allocated_range = fetch_range_from_zk()  // [1B, 1B+1M]
      self.pointer = 0
      self.local_codes = shuffle(convert_range_to_base62())
    
    def next_code(self):
      if self.pointer >= len(self.local_codes):
        self.allocated_range = fetch_range_from_zk()  // refill
        self.pointer = 0
      return self.local_codes[self.pointer++]

Benefits:
  • Zero collisions (counter is source of truth)
  • No database lookups (no race conditions)
  • Fast (purely in-memory shuffled list)
  • Stateless local generation
  • Graceful under write storms (just allocate new ranges)
```

### Collision Handling at Write Time (Critical)

Even with a collision-free ID generator, **custom aliases** may collide:

```
Write path collision check:
  1. Generate short_code (or user provided custom alias)
  2. Check if code exists:
     SELECT short_code FROM urls WHERE short_code = ?
  3. If exists:
     a) Return 409 Conflict OR
     b) Auto-retry with new code (transparent retry)
  4. Insert into urls table with:
     PRIMARY KEY (short_code)  // Unique constraint

Race condition scenario:
  Thread A: checks availability → not found
  Thread B: checks availability → not found
  Thread A: inserts → success
  Thread B: inserts → UNIQUE CONSTRAINT VIOLATION
  
Fix: Rely on database UNIQUE constraint; catch duplicate key exception
     and retry at application level.
```

---

## 6. Component 2: Write Path (Shorten)

### Request Processing Pipeline

```
POST /shorten
{
  "long_url": "https://example.com/very/long/path?a=1&b=2",
  "custom_code": "obama"  // optional
}

┌─────────────────────────────────────────┐
│ Step 1: Input Validation                │
│ • URL format check (HTTP/HTTPS)         │
│ • URL length limit (5KB max)            │
│ • Custom code format (alphanumeric only)│
│ • Virus/malware check (optional)        │
└──────────────────┬──────────────────────┘
                   │
┌──────────────────▼──────────────────────┐
│ Step 2: Uniqueness Check (optional)     │
│ • Query: URL already shortened?         │
│ • Cache: url_hash → existing short_code │
│ • Return existing code if found (idempotent)
└──────────────────┬──────────────────────┘
                   │
┌──────────────────▼──────────────────────┐
│ Step 3: Generate/Validate Short Code    │
│ • Custom code? Check availability       │
│ • Auto-generated? Get next from pool    │
│ • Collision check via UNIQUE constraint │
└──────────────────┬──────────────────────┘
                   │
┌──────────────────▼──────────────────────┐
│ Step 4: Insert into Database            │
│ INSERT INTO urls (short_code, long_url, │
│   created_at, user_id, expires_at)      │
│ ON CONFLICT → retry or return error     │
└──────────────────┬──────────────────────┘
                   │
┌──────────────────▼──────────────────────┐
│ Step 5: Cache & Return                  │
│ SET Redis short_code → {                │
│   long_url, created_at, user_id,        │
│   ttl: 24h                              │
│ }                                       │
│ Return { short_code, short_url }        │
└──────────────────────────────────────────┘
```

### Idempotency

If a user submits the same `long_url` twice, should they get the same `short_code`?

**Pro:** Reduces storage; user experience (same URL = same short code)
**Con:** Reveals who shortened the same URL; complicates analytics (multiple users on one code)

**Recommendation:** Implement via idempotency key (user provides one); if same key seen before, return prior result.

```
Request:
  POST /shorten
  Idempotency-Key: abc123
  { "long_url": "https://example.com/page" }

Database schema:
  CREATE TABLE url_writes (
    idempotency_key TEXT PRIMARY KEY,
    user_id TEXT,
    short_code TEXT,
    created_at TIMESTAMP
  );

Duplicate detection:
  1. Hash(user_id + idempotency_key) → lookup in url_writes
  2. Found? Return cached short_code
  3. Not found? Generate new, store both tables
```

### Write Latency Budget

For P99 < 500ms:

| Component | Budget |
|---|---|
| Network + validation | 50ms |
| Uniqueness check (cache) | 10ms |
| ID generation | 5ms |
| Database insert | 100ms |
| Replication lag (secondary acks) | 100ms |
| Cache write | 20ms |
| Network return | 50ms |
| **Total** | **335ms** |

**Headroom:** 165ms for GC pauses, network jitter, database load spikes.

---

## 7. Component 3: Custom URL Aliases

Vanity codes like `bit.ly/obama` are valuable but introduce complexity.

### Problem: Namespace Management

```
Scenarios:
  1. User A requests: custom_code = "obama"
     → Granted (available)
  2. User B requests: custom_code = "obama"
     → Denied (taken) or queued (wait-list)
  3. User A deletes their shortened URL
     → Is "obama" now available to User B?

Tradeoffs:
  • Reserved: High-value codes reserved by admin (CEO, brand names)
  • First-come-first-served: User A owns it until manual deletion
  • Expiring: Code available if URL not accessed in N days
  • Transfer: Allow users to "buy" codes from others (payment system)
```

### Implementation

```sql
CREATE TABLE custom_aliases (
  custom_code TEXT PRIMARY KEY,
  short_code TEXT NOT NULL UNIQUE,  -- FK to urls.short_code
  user_id TEXT,
  reserved BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP,
  deleted_at TIMESTAMP NULLABLE,
  
  FOREIGN KEY (short_code) REFERENCES urls(short_code)
);

Lookup on shortening request:
  SELECT * FROM custom_aliases
  WHERE custom_code = ? AND deleted_at IS NULL
  
  IF EXISTS:
    IF reserved=TRUE:
      Return 403 Forbidden (reserved)
    ELSE:
      Return 409 Conflict (user_id owns this code)
  
  IF NOT EXISTS:
    INSERT new custom_alias
    (Relies on UNIQUE(custom_code) constraint)
```

### Collision Under Concurrency

```
Race condition:
  User A: requests custom_code = "obama"
  User B: requests custom_code = "obama" (simultaneously)
  
  Both hit DB, both see "available", both attempt INSERT
  → Database UNIQUE constraint triggers on one of them
  → One succeeds, one fails

Prevention:
  a) Database UNIQUE constraint (standard)
  b) Distributed lock: Redis SET NX before INSERT
     SET obama:lock <user_id> NX EX 10 seconds
     If lock acquired → proceed; else → retry
```

### Staff-Level Concern: Reserved Word Management

Pre-allocate reserved codes for:
- Banned domains (phishing, malware)
- Company URLs (twitter, facebook, google)
- Numerically ambiguous codes (i, l, o = 1, L, 0 confusion)

```
Implementation:
  CREATE TABLE reserved_codes (
    code TEXT PRIMARY KEY,
    reason ENUM('phishing', 'company', 'ambiguous'),
    created_at TIMESTAMP
  );
  
  On custom code request:
    → Check against reserved_codes first
    → If found → reject immediately
```

---

## 8. Component 4: Read Path (Redirect)

The read path is the critical section: 100M+ redirects/day, P99 < 100ms.

### Redirect Pipeline (Cache-First)

```
GET /{short_code}

┌──────────────────────────────────────────┐
│ Step 1: Check Redis Cache                │
│ GET short_code from Redis                │
│ If HIT → deserialize and jump to Step 4  │
│ If MISS → proceed                        │
└──────────────────┬───────────────────────┘
                   │
┌──────────────────▼───────────────────────┐
│ Step 2: Query Primary Database           │
│ SELECT long_url FROM urls                │
│ WHERE short_code = ? AND active = TRUE   │
│ AND (expires_at IS NULL OR expires_at >  │
│   NOW())                                 │
└──────────────────┬───────────────────────┘
                   │
        ┌──────────┴──────────┐
        │                     │
    [Found]             [Not Found]
        │                     │
        ▼                     ▼
   Step 3:                Return 404
   Cache Hit                  │
        │                     │
        │            ┌────────▼──────────┐
        │            │ Log analytics     │
        │            │ (click tracking)  │
        │            │ → Kafka topic     │
        │            └────────┬──────────┘
        │                     │
        └─────────┬───────────┘
                  │
        ┌─────────▼──────────────────────┐
        │ Step 4: Return Redirect         │
        │ HTTP 301 (permanent)            │
        │ Location: {long_url}            │
        │ Cache-Control: public, max-age  │
        │ Log click event                 │
        │ → Kafka (async)                 │
        └─────────┬──────────────────────┘
                  │
        ┌─────────▼──────────────────────┐
        │ Step 5: Update Cache Locally    │
        │ SET Redis:                      │
        │   {short_code} → {long_url}     │
        │   TTL: 24 hours                 │
        │ (Lazy population; warm cache    │
        │  for future requests)           │
        └────────────────────────────────┘
```

### Database Query Optimization

The read query is simple but must be fast:

```sql
CREATE TABLE urls (
  short_code VARCHAR(10) PRIMARY KEY,
  long_url TEXT NOT NULL,
  user_id VARCHAR(50),
  created_at TIMESTAMP,
  expires_at TIMESTAMP NULLABLE,
  active BOOLEAN DEFAULT TRUE,
  created_count INT DEFAULT 0,  -- for analytics
  
  INDEX idx_user_id (user_id),
  INDEX idx_created_at (created_at),
  INDEX idx_expires_at (expires_at)
);

Query:
  SELECT long_url FROM urls
  WHERE short_code = ?
  AND active = TRUE
  AND (expires_at IS NULL OR expires_at > NOW())
  
Execution plan:
  • Primary key lookup on short_code (B-tree) → O(log N)
  • Filter on active, expires_at → done in-memory
  • Result: typically 1 row
  
Expected: < 1ms on cached page, ~5-10ms on disk hit
```

### Cache-Aside (Lazy Loading)

On first request to a URL:
1. Cache miss
2. Fetch from database
3. Populate cache (TTL: 24h)
4. Serve redirect

Subsequent requests hit Redis (sub-ms).

**Problem:** If a URL is edited (long_url changed), cache becomes stale.

**Solution:**
- Short TTL (24h) accepts some staleness
- Manual invalidation: on /edit endpoint, delete from cache immediately
- Stale-while-revalidate: return cached data while refreshing in background

```
Edit path:
  PUT /urls/{short_code}
  { "long_url": "https://newurl.com" }
  
  1. Update database
  2. DELETE short_code from Redis (invalidate cache)
  3. Return 200 OK
  
  Next GET request:
  • Cache miss → fetch new URL from DB
  • Populate cache
```

### HTTP Semantics

```
Redirect status codes:

301 (Moved Permanently):
  • Browser caches redirect locally
  • Second request goes directly to long_url (skips us)
  • Pro: Load reduction; user experience (faster)
  Con: Can't track analytics on every redirect
  
302 (Found, Temporary Redirect):
  • Browser doesn't cache
  • Every request comes to us (trackable)
  • Pro: Full analytics; can change target
  Con: Higher load on us; slower for user
  
307 (Temporary Redirect, HTTP/1.1):
  • Like 302 but preserves request method
  • POST → POST (vs 302 which changes to GET)

Recommendation:
  • Default: 301 (most users expect permanent)
  • Configurable: allow user to choose 301 vs 302
  • Analytics: track via server-side Kafka event, not HTTP redirects
```

### Client-Side Caching

Browsers cache 301 redirects aggressively, potentially ignoring your cache headers. If you need to invalidate a 301 redirect, you must:

1. Change HTTP status to 302 (temporary)
2. Serve fresh redirect with new target
3. Users already cached the 301 → won't see your update

**Mitigation:** Set a reasonable expires_at TTL in the database; after expiry, return 410 Gone.

---

## 9. Component 5: Caching Layer

### Cache Topology

```
Tier 1: CDN (CloudFlare, Akamai)
  • Caches HTTP redirects near geographic edges
  • Handles 301 responses
  • Pro: Ultra-low latency (~10ms globally)
  Con: Stateless; can't do custom logic
  
Tier 2: Local Redis (in-region)
  • Hot URL set (top 1% of URLs = 95% of reads)
  • Average TTL: 24 hours
  • Size: ~250 MB
  • Hit rate: 95%+
  
Tier 3: Database (primary + replicas)
  • Full URL store
  • Write-through; write-behind (async to replicas)
  • Multi-region replication
```

### Cache Warming

When a URL is shortened, should we warm the cache immediately?

**Option A: Eager warming**
```
On POST /shorten:
  1. Insert database
  2. SET Redis short_code → long_url (immediate)
  3. Return response

Benefit: Cache always warm for new URLs
Cost: Extra Redis operation (not critical path)
```

**Option B: Lazy loading (cache-aside)**
```
On GET /{short_code}:
  1. Check Redis (miss)
  2. Query database
  3. Populate Redis on first read
  4. Return redirect

Benefit: No extra work for URLs never accessed
Cost: First user sees longer latency (5-10ms vs <1ms)
```

**Recommendation:** Option A is better for short-term high-profile links. Option B is fine for the long tail. Use Option A if you can afford the Redis write.

### Cache Invalidation

> "There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

**Scenarios:**

| Event | Action |
|---|---|
| URL shortened | Warm cache (Option A) |
| URL edited | Invalidate Redis; DB update |
| URL deleted/disabled | Invalidate Redis; mark active=FALSE in DB |
| URL expires | DB query checks expires_at; no special cache action needed (lazy eviction) |
| Analytics updated | Separate cache; analytics doesn't affect redirect behavior |

**Staff-level consideration:** Cache invalidation can create race conditions if not careful.

```
Race condition (cache stampede):
  1. URL edit triggers cache invalidation (DEL key)
  2. Requests arrive between invalidation and DB update
  3. Cache miss → fetch stale URL from DB (if still cached there)
  4. Or, if DB update committed before query → fetch correct URL
  5. Result: Transient inconsistency

Mitigation:
  • Write-through: Update DB first, then invalidate cache
    (Between DB commit and cache invalidate, new requests see stale cache,
     which is safer than seeing wrong data)
  • Or: Versioning — store version number in cache; check on read
```

---

## 10. Component 6: Analytics & Tracking

### Click Tracking at Scale

Tracking 100M+ clicks/day requires careful design.

### Approach 1: Synchronous Updates (Don't do this)

```
GET /{short_code}
  1. Query URL
  2. UPDATE urls SET click_count = click_count + 1
  3. Return redirect

Problem:
  • Read path now includes a write lock
  • High contention on hot URLs (1M clicks/day → 1M locks)
  • Latency increases; throughput drops
  • Hot-key bottleneck: single URL dominates
```

### Approach 2: Async via Kafka (Standard)

```
GET /{short_code}
  1. Query URL (read-only; no lock)
  2. Publish to Kafka: { short_code, user_ip, referrer, user_agent, timestamp }
  3. Return redirect (latency unaffected)
  
Kafka consumer (analytics pipeline):
  1. Consume click events
  2. Aggregate: { short_code, count } per minute
  3. Write to analytics database (ClickHouse, BigQuery)
  4. Periodically sync back to main URL table
```

**Benefits:**
- Redirect latency unaffected (Kafka publish is fire-and-forget)
- Decoupled from analytics processing
- Can replay events if analytics pipeline bugs found

### Approach 3: Local Aggregation + Periodic Flush

```
In-memory aggregator (per worker):
  {
    "abc123": 1250,
    "xyz789": 834,
    ...
  }

Every 10 seconds:
  1. Flush aggregator to Kafka
  2. Reset aggregator

Benefit: Fewer Kafka publishes; lower network overhead
Downside: Data loss if worker crashes (1M clicks × 10s window = 10M lost)

Usually acceptable: Tolerate small data loss for analytics; re-query from logs if needed.
```

### Analytics Schema

```
Table: clicks (ClickHouse or BigQuery)
  short_code VARCHAR(10),
  long_url VARCHAR(5000),
  user_ip VARCHAR(39),      -- IPv6
  country VARCHAR(2),       -- GeoIP lookup
  referrer VARCHAR(2000),
  user_agent VARCHAR(1000),
  device_type ENUM('mobile', 'desktop', 'bot'),
  timestamp TIMESTAMP,
  
  PRIMARY KEY (short_code, timestamp)
  (Time-series table; partition by date)

Queries:
  SELECT COUNT(*) FROM clicks
  WHERE short_code = 'abc123'
    AND timestamp >= TODAY() - INTERVAL 7 DAY
  GROUP BY DATE(timestamp)
  → Retrieve click trend for past 7 days
  
  SELECT COUNT(*) FROM clicks
  WHERE timestamp >= NOW() - INTERVAL 1 HOUR
  GROUP BY country
  → Real-time geographic distribution
```

### The Hot-Key Problem (Staff-level)

A single shortened URL goes viral (e.g., election night, viral tweet). It receives 1M clicks in 1 hour.

```
Impact:
  • Write contention on single URL record in analytics DB
  • Kafka partition imbalance (all clicks route to one partition)
  • Redis cache hit rate stays high, but Kafka write becomes bottleneck
  
Detection:
  • Monitor click rate per short_code
  • Alert if any URL exceeds 10K clicks/minute
  
Mitigation:
  1. Local aggregation (buffer clicks in memory)
  2. Kafka partitioning strategy:
     Instead of:  partition = hash(short_code) % num_partitions
     Use:         partition = hash(user_ip + short_code) % num_partitions
     (Distribute same URL across multiple partitions)
  3. Throttle: Rate-limit clicks from same IP to same URL
     (Block bot attacks; legitimate users click once anyway)
```

---

## 11. Component 7: Storage Layer

### Multi-Tier Storage Design

| Layer | System | Purpose | Rationale |
|---|---|---|---|
| URLs (primary) | PostgreSQL / MySQL (replicat) | Current short codes, original URLs, metadata | ACID, strong consistency, replication |
| URLs (replica) | Read-only replicas in multiple regions | Geographic distribution for reads | Lower latency for global users |
| Custom aliases | Same as URLs | Namespace for vanity codes | Colocated for transactional consistency |
| Analytics | ClickHouse / BigQuery | Click events, trends, geolocation | OLAP workload; time-series optimized |
| Audit log | PostgreSQL / Cassandra | Who created/edited each URL | Compliance; user activity history |
| Cache | Redis | Hot URLs + metadata | Sub-ms lookups |

### Schema Design

```sql
CREATE TABLE urls (
  short_code VARCHAR(10) PRIMARY KEY,
  long_url TEXT NOT NULL,
  user_id VARCHAR(50) NOT NULL,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  expires_at TIMESTAMP NULLABLE,
  active BOOLEAN DEFAULT TRUE,
  deleted_at TIMESTAMP NULLABLE,
  
  -- Metadata
  title VARCHAR(500),                    -- User-provided title
  description VARCHAR(2000),             -- User-provided description
  tags VARCHAR(500),                     -- Comma-separated tags
  
  -- Analytics (denormalized; updated async)
  click_count INT DEFAULT 0,
  last_clicked_at TIMESTAMP NULLABLE,
  
  INDEX idx_user_id (user_id),
  INDEX idx_created_at (created_at),
  INDEX idx_expires_at (expires_at),
  UNIQUE(short_code)
);

CREATE TABLE custom_aliases (
  custom_code VARCHAR(100) PRIMARY KEY,
  short_code VARCHAR(10) NOT NULL,
  user_id VARCHAR(50) NOT NULL,
  reserved BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP DEFAULT NOW(),
  deleted_at TIMESTAMP NULLABLE,
  
  FOREIGN KEY (short_code) REFERENCES urls(short_code) ON DELETE CASCADE,
  INDEX idx_user_id (user_id),
  UNIQUE(custom_code, deleted_at)  -- Allow reuse after deletion
);

CREATE TABLE url_audit_log (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  short_code VARCHAR(10),
  user_id VARCHAR(50),
  action ENUM('create', 'update', 'delete', 'disable'),
  old_value TEXT,
  new_value TEXT,
  timestamp TIMESTAMP DEFAULT NOW(),
  
  INDEX idx_short_code (short_code),
  INDEX idx_user_id (user_id),
  INDEX idx_timestamp (timestamp)
);
```

### Partitioning Strategy (For Sharding)

As the database grows (billions of URLs), partition by `short_code` hash:

```
Partition key: hash(short_code) % num_shards

Shard assignment:
  Shard 0: hash(short_code) % 16 == 0
  Shard 1: hash(short_code) % 16 == 1
  ...
  Shard 15: hash(short_code) % 16 == 15

Benefits:
  • Uniform distribution (each shard gets ~1/16 of URLs)
  • Locality (all queries for a URL go to same shard)
  • Straightforward scaling (add shards; rebalance)

Lookup:
  shard_id = hash(short_code) % num_shards
  connection = get_connection_to_shard(shard_id)
  result = connection.query("SELECT * FROM urls WHERE short_code = ?")
```

### Replication & Failover

```
Setup: Multi-master replication across regions

Primary (US-EAST):
  • Handles all writes globally (single writer prevents conflicts)
  • Replicates changes to replicas

Replicas (US-WEST, EU, APAC):
  • Read-only; serve local reads
  • Replication lag: typically < 100ms

On primary failure:
  1. Detect via heartbeat timeout (10–30s)
  2. Promote US-WEST replica to primary
  3. Drain writes; switch writer endpoint
  4. Restart failed primary as replica
  
Tradeoff: Replication lag means stale reads during write storms.
Acceptable: Analytics data is eventually consistent anyway.
```

---

## 12. CAP Theorem Positioning

At Staff level, explicitly address CAP for each component:

| Component | CAP Choice | Reasoning |
|---|---|---|
| URL store (primary) | **CP** | Must be consistent; availability sacrificed during partition. A down database is worse than slow writes. |
| URL store (replicas) | **AP** | Replicas can lag; serving stale reads acceptable for a redirect service. |
| Cache (Redis) | **AP** | Stale cache is acceptable (TTL bounds staleness). Availability preferred over consistency. |
| Analytics (Kafka + warehouse) | **AP** | Eventual consistency okay. Lost clicks < lost redirects. |
| Custom aliases | **CP** | Namespace collision must be prevented; require strong consistency for deduplication. |

**PACELC implications:**
- **Consistent fast:** Can't have both at global scale (physics of network latency)
- **Consistent latent:** Replicate synchronously → adds latency
- **Available latent:** Replicate asynchronously → introduces staleness
- **Choice:** Available + latent (replicate async; tolerate stale reads)

---

## 13. Failure Modes & Mitigations

| Failure | Symptom | Detection | Mitigation |
|---|---|---|---|
| Database primary down | All writes fail; reads from replicas | Connection timeout; error spike | Failover to replica; promote to primary |
| Redis cache down | Cache misses spike; DB load increases | Cache error rate; hit rate drops | Switch to cache-aside; DB absorbs load |
| Collision storm | User gets 409 repeatedly on shorten | INSERT failure spike; custom code requests stuck | Auto-generate fallback code; exponential backoff |
| Hot-key bottleneck | One URL dominates; analytics DB slow | Click rate per URL > threshold | Partition Kafka by (IP + URL); local aggregation |
| ID generation pool exhausted | No new codes available (after 56B codes) | Generator returns error | Expand key space (7 chars → 8 chars); migrate live traffic |
| Clock skew | URLs expire at wrong time; scheduling goes wrong | System time drifts; NTP errors | Use NTP; logical clocks for distributed timestamps |
| Cascading deletes | Deleting one URL cascades to aliases/logs | Foreign key issues; data loss | Soft delete (deleted_at flag); no cascade |
| Cache stampede (crowd effect) | One URL expires in cache; 1000 requests flood DB | Sudden DB load spike; latency jump | Use probabilistic early expiration; stale-while-revalidate |
| Analytics lag | Click counts stale by hours | Kafka consumer lag monitoring | Scale consumers; add partitions |
| Duplicate shortened requests | Two users shorten same long URL | Depends on idempotency strategy | Idempotency key + dedup table |
| IP ban | Service blocked by CDN / upstream | 403 errors spike; geo concentrated | Rotate IPs; reduce request rate; use legitimate user-agent |

---

## 14. Concurrency & Collision Handling

### Race Condition: Simultaneous Custom Code Requests

```
User A                              User B
────────                            ────────
POST /shorten
{ custom_code: "obama" }
  │
  ├─ SELECT custom_code WHERE code='obama'
  │   → NOT FOUND
  │
  │                                 POST /shorten
  │                                 { custom_code: "obama" }
  │                                   │
  │                                   ├─ SELECT custom_code...
  │                                   │   → NOT FOUND
  │                                   │
  ├─ INSERT custom_aliases (obama, user_a)
│   → SUCCESS
│
│                                   ├─ INSERT custom_aliases (obama, user_b)
│                                   │   → UNIQUE CONSTRAINT VIOLATION
│                                   │   (Database rejects)
│                                   │
│                                   ├─ Catch exception
│                                   ├─ Retry with fallback code
│                                   ├─ INSERT custom_aliases (obama_2, user_b)
│                                   │   → SUCCESS

Prevention mechanism:
  • Database UNIQUE constraint on custom_code
  • Application-level retry with exponential backoff
  • Fallback: Auto-generate if custom code unavailable
```

### Compare-and-Swap (CAS) for Metadata Updates

If two users edit the same URL simultaneously:

```
User A                              User B
────────                            ────────
GET /urls/{short_code}
← Returns:
  { long_url: "...", version: 10 }

                                    GET /urls/{short_code}
                                    ← Returns:
                                      { long_url: "...", version: 10 }

PUT /urls/{short_code}
{ long_url: "new_url_a", version: 10 }
  │
  ├─ UPDATE urls
      SET long_url = 'new_url_a', version = 11
      WHERE short_code = ? AND version = 10
  │   → 1 row affected ✓

                                    PUT /urls/{short_code}
                                    { long_url: "new_url_b", version: 10 }
                                      │
                                      ├─ UPDATE urls
                                          SET long_url = 'new_url_b', version = 11
                                          WHERE short_code = ? AND version = 10
                                      │   → 0 rows affected ✗
                                      │
                                      ├─ Return 409 Conflict
                                      ├─ User B retries:
                                      ├─ GET /urls/{short_code}
                                      │   ← version is now 11
                                      ├─ PUT with version: 11
                                      │   → 1 row affected ✓

Result: User A's edit wins; User B sees conflict and retries.
```

### Collision Probability Redux

For 1B URLs using 6-character base-62 codes (56B possible):

**Birthday paradox:**
```
P(at least one collision) ≈ (1B)² / (2 × 56B) ≈ 0.009 ≈ 0.9%
Expected collisions: ~10M

For 10B URLs: P ≈ 0.09 ≈ 9%
For 56B URLs: P ≈ 100% (guaranteed collision)
```

**Staff insight:** At 100K new URLs/day, you'll hit 1B URLs in ~27 years. Plan for 7-character codes (3.6T possible) within 10 years to stay safe.

---

## 15. Global Distribution & Geo-Routing

### Multi-Region Deployment

```
US-EAST (primary write region)
  ├─ Master database
  ├─ Primary Redis
  ├─ Write API
  └─ Replication to other regions
  
US-WEST (replica region)
  ├─ Read-only database replica
  ├─ Local Redis replica
  ├─ Read API (via replica)
  └─ Can be promoted to primary on US-EAST failure

EU (replica region)
  ├─ Read-only database replica (lag: ~50–100ms)
  ├─ Local Redis replica
  ├─ Read API (via replica)

APAC (replica region)
  ├─ Same structure

CDN (CloudFlare, Akamai)
  ├─ Caches HTTP 301 redirects
  ├─ Served from ~200+ edge locations globally
  ├─ TTL: 24 hours (or configurable)
```

### Routing Logic

```
Shorten request (write):
  Route to US-EAST primary
  (Strong consistency required)
  
Redirect request (read):
  1. Check CDN cache (hits on hot URLs)
  2. If miss: Route to nearest region
     (User in Europe → query EU replica)
  3. Fallback: If all replicas down, fallback to primary

Detection:
  Health checks on replicas every 10s
  If replica latency > 500ms or error rate > 1%:
    Fallback to primary (higher latency but available)
```

### Replication Lag Handling

```
Scenario: User shortens URL in US-EAST; immediately travels to Europe;
          tries to redirect from EU replica before replication completes.

Timeline:
  T=0:     Write to US-EAST → success
  T=0:     Replication begins (async)
  T=10ms:  User in EU makes redirect request
  T=10ms:  EU replica query → URL not found (replication lag ~50ms)
  
Mitigation:
  1. Cache write-through: on POST /shorten, immediately populate local Redis
  2. "Write your own read": after shortening, wait for Redis replication
  3. Accept stale reads: EU user sees 404; retry → succeeds in 50ms
```

---

## 16. Senior vs Staff Answer Differentiators

### Senior-level expectations

- Identify major components (shorten API, redirect API, database, cache)
- Explain basic unique ID generation (random + collision check)
- Discuss HTTP 301 vs 302 semantics
- Handle obvious failures (database down → failover)
- Provide basic capacity math
- Mention analytics (async tracking)

### Staff-level expectations (additional)

| Topic | What Staff-level adds |
|---|---|
| ID Generation | Hybrid counter + shuffle approach; collision probability math; distribution challenges |
| Custom aliases | Namespace management; race conditions; reserved word handling |
| Write path | Idempotency keys; write latency budget; double-write prevention |
| Read path | Cache-aside strategy; stale-while-revalidate; 301 caching implications |
| Analytics | Hot-key detection and mitigation; local aggregation; Kafka partitioning by (IP + code) |
| Caching | Cache stampede; probabilistic expiration; cache invalidation race conditions |
| Concurrency | CAS versioning; UNIQUE constraints as correctness mechanism; retry semantics |
| Global distribution | Multi-master replication lag; fallback strategies; replica consistency model |
| CAP theorem | Explicit CP vs AP decisions; PACELC reasoning for each component |
| Collision handling | 6-char code space saturation; long-term planning (7-char migration); birthday paradox math |
| Sharding | Partition by short_code hash; hot-key detection; rebalancing strategy |
| Failure modes | Clock skew, cascading deletes, collision storms, IP bans, cache stampede |
| Proactive signals | Surface hot-key problem, replication lag, collision saturation risk before being asked |

### The single most important Staff differentiator

**Collision management at scale.** A Senior engineer designs for 1B URLs. A Staff engineer designs for 10B URLs and explains when the 6-character space becomes saturated. They proactively discuss the 7-character migration path, the cost of rebalancing, and how customer-facing short codes need versioning to survive the transition.

---

## 17. Interview Time Allocation

For a 45-minute system design session:

| Phase | Time | Focus |
|---|---|---|
| Requirements & scope | 5 min | Write: 100K/day; Read: 100M/day; P99 latency < 100ms |
| Capacity estimation | 5 min | Throughput, storage, collision probability, cache math |
| High-level architecture | 5 min | Shorten API, redirect API, database, cache, analytics (Kafka) |
| Unique ID generation (deep dive) | 10 min | Counter + shuffle vs random + retry; collision math; saturation |
| Write path | 7 min | Idempotency, custom aliases, UNIQUE constraint handling |
| Read path | 8 min | Cache-aside, 301 semantics, stale-while-revalidate |
| Caching strategy | 5 min | Redis topology, TTL, cache stampede, invalidation |
| Analytics | 5 min | Kafka, hot-key detection, local aggregation |
| Global distribution | 3 min | Multi-region setup, replication lag, failover |
| Failure modes | 3 min | Top 5: database down, collision storm, hot-key, cache stampede, clock skew |

**What to cut if short on time:** Global distribution, analytics detail. **Never cut:** ID generation, collision handling, or read-path caching.

---

## 18. Quick-Reference Cheatsheet

```
KEY NUMBERS
───────────
100K shorts/day       → 1.16 URLs/sec (write) — trivial
100M redirects/day    → 1,150 redirects/sec (read) — cache-critical
6-char base-62        → 56 billion possible codes
Collision @ 1B URLs   → 0.9% (10M collisions)
Collision @ 10B URLs  → 9% (UNSUSTAINABLE)

CAPACITY
────────
Per URL storage:      2.5 KB
Annual growth (100K/day): 91 GB
10 years:             910 GB
Hot cache (1% URLs):  250 MB Redis
Cache hit rate:       95%+

ARCHITECTURE DECISIONS
──────────────────────
Write path:    Shorten API → DB write → Redis populate (eager)
Read path:     GET → Redis check → DB fallback → 301 redirect → Kafka click event
Analytics:     Async via Kafka (fire-and-forget; decouple from critical path)
Caching:       Cache-aside with 24h TTL; stale-while-revalidate on expiry
Replication:   Async multi-region replicas; CP for primary, AP for replicas

ID GENERATION STRATEGIES
────────────────────────
Random + Retry:       P(collision) checks; requires DB lookup; O(1) expected retries
Counter + Base-62:    Sequential; predictable; O(1) guaranteed no collision
Counter + Shuffle:    RECOMMENDED: no collisions, fast, non-sequential locally

CUSTOM ALIASES
──────────────
Collision check:      UNIQUE constraint in database
Reserved codes:       Pre-populate table; check first on every request
Transfer/expiry:      Soft delete (deleted_at); allow re-use after N days

CAP DECISIONS
─────────────
Primary database:     CP (writes must be strong consistent)
Read replicas:        AP (stale acceptable; read latency priority)
Cache:                AP (stale acceptable; availability priority)
Aliases (namespace):  CP (no concurrent same-code writes)

FAILURE MODES (TOP 5)
────────────────────
1. Database primary down → Failover to replica; write buffering (queue)
2. Collision storm → Auto-generate fallback; exponential backoff
3. Hot-key bottleneck → Partition Kafka by (IP + code); local aggregation
4. Cache stampede → Probabilistic expiration; stale-while-revalidate
5. Clock skew → Use NTP + logical timestamps for scheduling

STAFF-LEVEL SIGNALS TO HIT
───────────────────────────
✓ Collision probability math (birthday paradox; saturation at 10B)
✓ 7-character code space migration path (future-proofing)
✓ CAS versioning for concurrent edits
✓ Hot-key detection + mitigation (local aggregation, Kafka partitioning)
✓ Cache stampede + probabilistic expiration
✓ Replication lag handling (write-your-own-read; fallback)
✓ Explicit CAP reasoning per component
✓ UNIQUE constraint as correctness mechanism
✓ Idempotency key strategy (avoid double-shortened URLs)
✓ Custom alias namespace management (reserved words, expiry)
✓ Proactive failure mode enumeration
```

---

*Guide compiled for Senior & Staff-level system design interview preparation.*
*Topics: URL Shortener, Distributed Systems, Caching, ID Generation, Collision Handling, CAP Theorem.*
