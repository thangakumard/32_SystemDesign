# URL Shortener — System Design Interview Guide
### Senior & Staff Engineer Level

---

## Table of Contents

1. [Problem Statement & Scope](#1-problem-statement--scope)
2. [Requirements](#2-requirements)
3. [Capacity Estimation](#3-capacity-estimation)
4. [Core Entities](#4-core-entities)
5. [APIs](#5-apis)
6. [High-Level Architecture](#6-high-level-architecture)
7. [Component 1: Unique ID Generation](#7-component-1-unique-id-generation)
   - [Approach 1: Distributed Snowflake-Style ID (Senior-level)](#approach-1-distributed-snowflake-style-id-senior-level)
   - [Approach 2: Hash-Based Generation (more common in practice)](#approach-2-hash-based-generation-more-common-in-practice)
   - [Approach 3: Counter-Based with Base Conversion (Staff-level insight)](#approach-3-counter-based-with-base-conversion-staff-level-insight)
   - [Recommended Approach: Hybrid Counter + Shuffle](#recommended-approach-hybrid-counter--shuffle)
8. [Component 2: Write Path (Shorten)](#8-component-2-write-path-shorten)
9. [Component 3: Custom URL Aliases](#9-component-3-custom-url-aliases)
10. [Component 4: Read Path (Redirect)](#10-component-4-read-path-redirect)
11. [Component 5: Caching Layer](#11-component-5-caching-layer)
12. [Component 6: Analytics & Tracking](#12-component-6-analytics--tracking)
13. [Component 7: Storage Layer](#13-component-7-storage-layer)
14. [CAP Theorem Positioning](#14-cap-theorem-positioning)
15. [Failure Modes & Mitigations](#15-failure-modes--mitigations)
16. [Concurrency & Collision Handling](#16-concurrency--collision-handling)
17. [Global Distribution & Geo-Routing](#17-global-distribution--geo-routing)
18. [Senior vs Staff Answer Differentiators](#18-senior-vs-staff-answer-differentiators)
19. [Interview Time Allocation](#19-interview-time-allocation)
20. [Quick-Reference Cheatsheet](#20-quick-reference-cheatsheet)
21. [Scaling Reads to 10K Redirects/Second](#21-scaling-reads-to-10k-redirectssecond)

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

## 4. Core Entities

The nouns the system stores and passes around. API contracts (next section) and the storage schema ([Storage Layer](#13-component-7-storage-layer)) are built from these.

| Entity | Key Fields | Notes |
|---|---|---|
| **URL** | `short_code` (PK), `long_url`, `user_id`, `created_at`, `updated_at`, `expires_at`, `active`, `version` | The mapping record. `short_code` is the primary key; `version` enables CAS on edits. |
| **CustomAlias** | `custom_code` (PK), `short_code`, `user_id`, `reserved`, `deleted_at` | Vanity codes; separate namespace so reserved words and reuse-after-delete can be managed. |
| **User** | `user_id` (PK), `api_key`, `plan`, `rate_limit` | Owner of URLs; anonymous creation is optional. |
| **URLAuditLog** | `id` (PK), `short_code`, `user_id`, `action`, `old_value`, `new_value`, `timestamp` | Append-only history of create/update/delete/disable; written on every change to a URL. Edits update `urls` in place, so this is where past versions live. |
| **ClickEvent** | `short_code`, `timestamp`, `ip`, `country`, `referrer`, `user_agent`, `device_type` | Append-only analytics fact; written via Kafka, not on the redirect path. |

```
User 1 ──── * URL 1 ──── 0..1 CustomAlias
                │
                ├──── * URLAuditLog  (one row per create/edit/delete/disable)
                │
                └──── * ClickEvent   (analytics store, not the primary DB)
```

---

## 5. APIs

REST over HTTPS. Writes and management require authentication (API key / OAuth token); redirects are public.

### Endpoints

| Operation | Method & Path | Auth | Purpose |
|---|---|---|---|
| Shorten | `POST /shorten` | Yes | Create a short URL (optionally with a custom code, expiry) |
| Redirect | `GET /{short_code}` | No | Resolve short code and redirect to the original URL |
| Edit | `PUT /urls/{short_code}` | Yes (owner) | Change target URL or expiry; requires `version` for CAS |
| Disable / delete | `DELETE /urls/{short_code}` | Yes (owner) | Soft delete: sets `active=false`, keeps the record |
| Analytics | `GET /stats/{short_code}?from=&to=&group_by=` | Yes (owner) | Click counts, geo, referrer, device breakdown |

### Contracts

```
POST /shorten
Idempotency-Key: abc123              // optional
{
  "long_url": "https://example.com/very/long/path?a=1&b=2",
  "custom_code": "obama",            // optional
  "expires_at": "2027-01-01T00:00:00Z",  // optional
  "redirect_type": 301               // optional: 301 | 302
}
→ 201 Created
{ "short_code": "obama", "short_url": "https://sho.rt/obama", "expires_at": "..." }

GET /{short_code}
→ 301/302  Location: {long_url}
→ 404 Not Found   (unknown code)
→ 410 Gone        (expired or disabled)

PUT /urls/{short_code}
{ "long_url": "https://newurl.com", "version": 10 }
→ 200 OK  { "version": 11 }
→ 409 Conflict    (stale version)

DELETE /urls/{short_code}
→ 204 No Content

GET /stats/{short_code}?from=2026-01-01&to=2026-01-07&group_by=day
→ 200 OK  { "total_clicks": 12345, "series": [...], "by_country": {...} }
```

### Status codes

| Code | Meaning |
|---|---|
| 201 | Short URL created |
| 301 | Permanent redirect: browsers cache it, so repeat visits skip the server (less load, but edits/expiry may not take effect for clients that cached it). Use only if the destination is truly permanent |
| 302 | Temporary redirect: not cached by default, so every click reaches the redirect layer (edits/expiry apply immediately). Use when the destination can change or expire; the usual choice for editable/expiring short links |

Choose 301 vs 302 based on whether the destination is truly permanent. Choose the analytics design separately: collect click events from the layer that actually serves the redirect (CDN access logs/edge events, or the redirect service's Kafka event).
| 400 | Invalid URL or custom code format |
| 403 | Reserved code, or not the owner |
| 404 | Short code not found |
| 409 | Custom code taken, or stale `version` on edit |
| 410 | Expired or disabled |
| 429 | Rate limit exceeded |

**Design notes:**
- The redirect endpoint sits at the root path (`/{short_code}`) to keep short URLs short; management APIs live under `/urls` and `/stats` to avoid colliding with codes.
- Pagination (cursor-based) applies to listing a user's URLs: `GET /urls?cursor=`.
- Rate limit `POST /shorten` per user/API key at the gateway; redirects are limited per IP only to blunt abuse.

---

## 6. High-Level Architecture

```
                     ┌──────────────────────┐
                     │  User / Client       │
                     │  (Web, Mobile, API)  │
                     └──────────┬───────────┘
                                │
                     ┌──────────▼───────────┐
                     │  CDN (edge cache)    │
                     │  hot 301/302 hits    │
                     └──────────┬───────────┘
                                │ miss / API calls
                     ┌──────────▼───────────┐
                     │ API Gateway / LB     │
                     │ TLS, auth, rate limit│
                     │ (routing only)       │
                     └───┬──────┬───────┬───┘
         POST /shorten   │      │       │  GET /stats/{id}
         PUT/DELETE /urls│      │       │
         (100K/day)      │      │ GET /{short_code}
                         │      │ (100M/day)
        ┌────────────────▼┐  ┌──▼───────────────┐  ┌──────────────────┐
        │ Write Service   │  │ Read Service     │  │ Analytics Service│
        └────────┬────────┘  └────────┬─────────┘  └────────┬─────────┘
                 │                    │                     │
   ┌─────────────▼────────────────────▼─────────────────────▼───────────┐
   │ Data & Messaging Tier (accessed only by the services above)        │
   │                                                                    │
   │  ┌───────────────┐  ┌───────────────┐  ┌───────────────────────┐   │
   │  │ ID Generator  │  │ Redis Cache   │  │ Primary Data Store    │   │
   │  │ ZK-backed     │  │ hot URLs      │  │ MySQL / PostgreSQL    │   │
   │  │ ranges        │  │ ~250MB, 24h   │  │ urls, custom_aliases, │   │
   │  │ → short_code  │  │               │  │ url_audit_log         │   │
   │  └───────────────┘  └───────────────┘  │ sharded by short_code │   │
   │                                        └──────────┬────────────┘   │
   │  ┌───────────────┐                     async repl. │                │
   │  │ Kafka         │                     ┌──────────▼────────────┐   │
   │  │ click events  │                     │ Regional Read Replicas│   │
   │  └───────┬───────┘                     │ (APAC, EU, AMER)      │   │
   │          │                             └───────────────────────┘   │
   │  ┌───────▼───────┐  ┌───────────────────────┐                      │
   │  │ Analytics     │─►│ ClickHouse / BigQuery │                      │
   │  │ Pipeline      │  │ (click store)         │                      │
   │  │ (Spark/Flink) │  └───────────────────────┘                      │
   │  └───────────────┘                                                 │
   └────────────────────────────────────────────────────────────────────┘
```

**Flows**
- **Redirect (core path):** `GET /{short_code}` → Gateway → Read Service → check Redis. **Hit:** return the long URL immediately. **Miss:** read the database (nearest read replica), store the mapping in Redis for later requests, then return the redirect. A click event is published to Kafka asynchronously.
- **Shorten:** Gateway → Write Service → ID Generator (get `short_code`) → Primary DB (insert) → return. *Optional:* also write to Redis (eager warming) for predicted hot links; see Cache Warming.
- **Edit/Disable:** Gateway → Write Service → Primary DB (update) → Redis (invalidate).
- **Stats:** Gateway → Analytics Service → ClickHouse / BigQuery.

**Kafka** decouples writes from analytics processing. Analytics can lag without impacting the critical path.

---

## 7. Component 1: Unique ID Generation

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
     (Note: base-62 = a number system with 62 symbols: a-z, A-Z, 0-9.
      URL-safe and compact; 6 chars → 62^6 ≈ 56.8B possible codes.)
  2. Check if code exists in database (via lookup or cache)
  3. If exists → retry with different candidate (exponential backoff)
  4. If doesn't exist → insert and return

Collision resolution:
  Random retry:  O(1) expected tries if collision rate is low
  Hash-based:    Deterministic; same inputs → same code (dedup possible); on collision, change the salt and re-hash
  
Downsides:
  • Requires database lookup before insertion (critical path overhead)
  • Race condition: two threads may generate same code simultaneously
  • Retry storms possible under high write concurrency

Retry policy:
  Max retries:  3-5 (cap at 5)
    • 6-char base-62 → ~56.8B codes; at 1B stored URLs, p(collision) ≈ 1.8% per try
    • All 3 tries fail ≈ 6 in a million; all 5 tries fail ≈ 2 in a billion
    • Hitting the cap signals a real problem (keyspace nearly full, DB issue), so fail instead of looping
  Backoff:      Not needed for pure collisions (each random candidate is independent → retry immediately)
    • Use small backoff + jitter only for contention/transient errors: 10ms, 20ms, 40ms (+ jitter)
  Timeouts:
    • Single DB/cache existence check: 20-50 ms
    • Whole write path (all retries):  200-500 ms budget
    • Client-facing request timeout:   1-2 s
  On budget/cap exceeded:
    • Return 503 Service Unavailable with Retry-After (not 409; the conflict is internal)
    • Log + alert; frequent retries >2 → increase code length (6 → 7 chars) or switch to counter approach
    • Idempotency-Key lets the client safely retry the whole request without duplicates

Better option: skip check-then-insert; just INSERT with a UNIQUE constraint and
retry only on constraint violation (removes the race condition and saves a round trip)
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

**60-second talk track:**

> "The third approach avoids collisions by construction instead of detecting them. I keep a single global counter, and every time a user shortens a URL I atomically increment it, so each request gets a unique integer. I then convert that integer to base-62, using a-z, A-Z and 0-9, which turns a number like one billion into a compact 6-character code. Because every integer is unique, every code is unique, so there's no database lookup before insert, no retries, and no race conditions. Generation is O(1), and 6 base-62 characters give us about 56 billion codes, which is plenty for our scale.
>
> The trade-offs are two things:
> - The counter is a single point of contention and failure, so in production I'd not hit it per request. Instead, each app server leases a batch, say a million IDs, from ZooKeeper or a database sequence and hands them out locally.
> - The codes are sequential, so users can guess the next one and infer our traffic volume. To fix that, I'd shuffle each batch locally, or use a bijective scramble of the integer before encoding.
>
> That's my recommended hybrid: counter-based uniqueness with non-guessable codes."

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

**60-90 second talk track:**

> "My recommended design is a hybrid: a counter for uniqueness plus a shuffle for unpredictability. A pure counter gives zero collisions and O(1) generation, but its codes are sequential, so anyone can guess the next one and estimate our traffic. A pure random or hash approach hides the pattern but needs a database check and retries.
>
> The hybrid takes the best of both. A central, durable counter, such as a Postgres sequence or ZooKeeper, doesn't hand out single IDs. It leases ranges. Each app server asks for a block, say one million IDs from one billion to one billion plus a million. The server converts that block to base-62 codes and shuffles them in memory. After that, creating a short URL is just popping the next code off the shuffled list. There's no database lookup, no collision check and no network call on the hot path.
>
> Uniqueness is guaranteed because ranges never overlap and base conversion is one-to-one. Codes look random because of the shuffle. When a server runs out, it fetches a new range, so even a write storm only adds a few cheap calls to the counter.
>
> The trade-offs:
> - If a server crashes, its unused IDs are lost, which is fine with a 56-billion keyspace.
> - The counter itself must be durable. ZooKeeper or the database sequence has to persist its value, with writes synced to disk and replicated, so that after a full cluster restart it resumes from where it left off instead of from zero. If it reset, it would reissue ranges that were already handed out and cause collisions with previously issued codes.
> - Custom aliases can still collide, so I enforce a UNIQUE constraint on the write path and keep the alias namespace separate from generated codes."

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

## 8. Component 2: Write Path (Shorten)

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
│ Step 5: Return (optional: warm cache)   │
│ Optional eager warming, hot links only: │
│ SET Redis short_code → {                │
│   long_url, created_at, user_id,        │
│   ttl: min(24h, expire_at - now) ± jitter│
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

## 9. Component 3: Custom URL Aliases

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

## 10. Component 4: Read Path (Redirect)

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
        │   TTL: min(24h, expire_at-now)  │
        │        ± 5-10% jitter           │
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

This is the main reason redirects are fast: `GET /{short_code}` checks Redis; a hit returns the long URL immediately, and a miss reads the database and stores the mapping in Redis for later requests.

On first request to a URL:
1. Cache miss
2. Fetch from database
3. Populate cache (TTL: `min(24h, expire_at - now)` ± 5-10% jitter; links with no expiry use 24h; if `ttl <= 0`, don't cache and return 410 Gone)
4. Serve redirect

Subsequent requests hit Redis (sub-ms).

**Problem:** If a URL is edited (long_url changed), cache becomes stale.

**Solution:**
- Short TTL (capped at 24h) accepts some staleness
- Cap TTL at the link's remaining lifetime (`min(24h, expire_at - now)`) so an expired link is never served from cache
- Add ±5-10% jitter to TTLs to avoid a cache stampede when many keys expire together
- Negative caching: cache 404/410 results for 30-60 s to protect the DB from repeated lookups
- Keep an `if now >= expire_at → 410` check at redirect time as a safety net (clock skew)
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
  Con: Can't change/expire target for clients that already cached it
       (cached repeat visits also won't reach the redirect service; see analytics note below)
  
302 (Found, Temporary Redirect):
  • Browser doesn't cache by default
  • Every request reaches the redirect layer (CDN or our service)
  • Pro: Can change/expire target immediately
  Con: Higher load on us; slightly slower for user
  
307 (Temporary Redirect, HTTP/1.1):
  • Like 302 but preserves request method
  • POST → POST (vs 302 which changes to GET)

Recommendation:
  • Default: 301 (most users expect permanent)
  • Configurable: allow user to choose 301 vs 302
  • Pick 301 vs 302 by whether the destination is truly permanent, not by analytics needs
  • Analytics is a separate decision: collect events from the layer serving the redirect
    - Browser-cached 301s never reach the CDN or our service (only the first visit per browser is observable)
    - CDN-cached redirects may not reach our service, but CDN access logs / edge events can still emit click events
    - Redirect service: publish a server-side Kafka click event (not tied to the HTTP status code)
```

### Client-Side Caching

Browsers cache 301 redirects aggressively, potentially ignoring your cache headers. If you need to invalidate a 301 redirect, you must:

1. Change HTTP status to 302 (temporary)
2. Serve fresh redirect with new target
3. Users already cached the 301 → won't see your update

**Mitigation:** Set a reasonable expires_at TTL in the database; after expiry, return 410 Gone.

### Choosing 301 vs 302 (and What the CDN May Cache)

Decide per link with two questions: can the destination change or expire, and do we need to see every request? The status code signals the first; what the CDN actually obeys is the `Cache-Control` header, so set both deliberately.

| Link type | Status | CDN cache (`s-maxage`) | Browser cache (`max-age`) |
|---|---|---|---|
| Truly permanent, never edited/expired | 301 | Long (e.g. days) | Long |
| May be edited, expires, or can be disabled | 302 | Short, capped at `min(cap, expire_at - now)`; purge on `PUT`/`DELETE` | Off or very short |
| Dynamic logic (geo/device targeting, A/B routing, per-request rate limits) | 302 | Don't cache at CDN; go to the service | Off |
| 404 / 410 (missing, expired, disabled) | 404 / 410 | Don't cache, or only briefly (30-60 s) | Off |

**Rules of thumb:**
- Permanent and stable → long CDN caching; most requests never reach the service.
- Changeable or expiring → short TTL plus explicit CDN purge on edit/delete.
- Dynamic or about to expire → serve from the service.
- Analytics is a separate decision: if the CDN serves the redirect, collect click events from CDN access logs/edge events, plus the service's Kafka event for requests that reach origin. Don't use the status code as the analytics switch.

**60-90 second talk track:**

> "I decide this per link with two questions: can the destination ever change or expire, and do I need to see every request? The status code expresses the first, but what the CDN actually obeys is the Cache-Control header, so I set both deliberately.
>
> A 301 means the owner told us the destination is truly permanent. Those are safe to cache at the CDN, and in browsers, for a long time, with a long max-age and s-maxage. Most requests never reach my service, which gives the biggest load reduction. I wouldn't use a 301 for anything with an expiry, a possible edit, or a disable flag.
>
> A 302 means the destination may change or expire. I can still let the CDN cache it, but with a short s-maxage capped at the remaining lifetime, min(cap, expire_at - now), and browser caching off or very short. Then edits and deletes take effect quickly, and I also purge the CDN key on PUT or DELETE.
>
> Some responses I don't cache at the CDN:
> - 404s, or only briefly, so a newly created code doesn't look missing.
> - 410s for expired or disabled links, or only briefly.
> - Links with per-request logic, like geo or device targeting, A/B routing or rate limiting.
> - Links that are about to expire.
>
> Analytics is a separate decision. If the CDN serves the redirect, my service won't see it, so I collect click events from the CDN's access logs or edge events, plus my own Kafka event for requests that reach origin. I don't use the status code as the analytics switch.
>
> So the rule is: permanent and stable means long CDN caching, changeable or expiring means short TTL plus purge on change, and anything dynamic goes to my service."

---

## 11. Component 5: Caching Layer

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

Redirects are fast because of cache-aside (Option B below), not because of warming. Eager warming is an optional optimization on top of it.

**Option A: Eager warming (optional)**
Happens at creation time: after saving the new short code to Postgres, also write it to Redis before anyone has requested it.
```
On POST /shorten:
  1. Insert database
  2. SET Redis short_code → long_url (immediate)
  3. Return response

Benefit: Avoids the first cache miss for that link
Cost: Extra Redis operation (not critical path); uses cache space for links that may never be opened
```

**Option B: Lazy loading (cache-aside, the default)**
```
On GET /{short_code}:
  1. Check Redis (miss)
  2. Query database
  3. Populate Redis on first read
  4. Return redirect

Benefit: No extra work for URLs never accessed
Cost: First user sees longer latency (5-10ms vs <1ms)
```

**Recommendation:** Use Option B (cache-aside) as the default for all links. Treat Option A as an optional optimization for predicted hot links (for example a campaign or launch link), not for every new URL.

### Cache Invalidation

> "There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

**Scenarios:**

| Event | Action |
|---|---|
| URL shortened | No cache action by default (populated on first read); optionally warm if predicted hot (Option A) |
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

## 12. Component 6: Analytics & Tracking

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

## 13. Component 7: Storage Layer

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
  version INT NOT NULL DEFAULT 1,        -- CAS counter; bumped on every edit
  
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

## 14. CAP Theorem Positioning

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

## 15. Failure Modes & Mitigations

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

## 16. Concurrency & Collision Handling

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

## 17. Global Distribution & Geo-Routing

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

## 18. Senior vs Staff Answer Differentiators

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

## 19. Interview Time Allocation

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

## 20. Quick-Reference Cheatsheet

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
Caching:       Cache-aside with TTL = min(24h, expire_at - now) + jitter; stale-while-revalidate on expiry
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

## 21. Scaling Reads to 10K Redirects/Second

**Question:** *How can your system scale to support 10K redirects (reads) per second?*

10K/s is about 9× the 1,150/s average in the capacity estimate (and close to the 11.5K/s peak). The write path (~1/s) is irrelevant here; this is purely a read-scaling problem. The workload is ideal for scaling: **read-only, key-value lookups, a tiny hot set (~250 MB), and tolerant of slight staleness.**

### The answer in one line

**Three layers, each absorbing traffic so the next sees far less:** the **CDN** absorbs edge traffic, **Redis** handles CDN misses, and **stateless Read Services scale horizontally behind a load balancer**. The database only sees Redis misses.

```
 10,000 req/s
      │
 ┌────▼─────┐  ~60% hit       ┌────────────────────────────────────────┐
 │   CDN    ├────────────────►│ served at edge (never reaches origin)  │
 └────┬─────┘                 └────────────────────────────────────────┘
      │ ~4,000 req/s
 ┌────▼───────────────────┐
 │ Load balancer (multi-AZ)│
 └────┬───────────────────┘
      │
 ┌────▼───────────────────────────────┐
 │ Read Service × 8–10 (stateless,    │──► Kafka (async, batched click events)
 │ autoscaled)                        │
 └────┬───────────────────────────────┘
      │ ~4,000 lookups/s
 ┌────▼──────────────────────────────┐
 │ Redis (cluster + replicas), 95%+ hit│
 └────┬──────────────────────────────┘
      │ ~200 reads/s (misses only)
 ┌────▼──────────────────────────────┐
 │ DB read replicas (shard if needed) │
 └────────────────────────────────────┘
```

Even in the worst case with **no CDN**, Redis sees 10K ops/s (one node handles ~100K) and the DB sees 5% = 500 reads/s, which a single replica handles.

### Core design: the three layers

#### 1. CDN: absorbs edge traffic

- Cache redirect responses at the edge (see [Choosing 301 vs 302](#choosing-301-vs-302-and-what-the-cdn-may-cache)). Permanent links get a long `s-maxage`; editable/expiring links get a short, capped `s-maxage` plus a purge on `PUT`/`DELETE`.
- A viral link at 100K clicks/s is still one cached object at the edge. Origin shield / request collapsing means 1,000 simultaneous requests for an uncached link become one origin request.
- Dynamic links (geo/device targeting, A/B) and 404/410s are not cached for long, so they still reach the service.

#### 2. Redis: handles CDN misses

- Cache-aside: the Read Service checks Redis, reads the DB on a miss, then populates Redis with TTL = `min(24h, expire_at - now)` ± jitter (see [Cache-Aside](#cache-aside-lazy-loading)).
- The hot set is ~250 MB, so the working set fits in memory on every node. The constraint is ops/s and availability, not capacity.
- One node does ~100K ops/s, so 4K–10K/s fits easily; use **Redis Cluster** (shard by `short_code`, replicas per shard) for headroom and failover, and read from replicas.
- On edit/delete: update the DB, then invalidate Redis (and purge the CDN key).
- Result: 95%+ hit rate, so the database sees ~200 reads/s. These are primary-key point reads (~1 ms in memory, 5–10 ms from disk) on regional read replicas; if data or miss traffic outgrows one primary, shard by `hash(short_code)` (see [Partitioning](#partitioning-strategy-for-sharding)).

#### 3. Stateless Read Services: scale horizontally behind a load balancer

- The service holds no state, so scaling is "add instances": round-robin / least-connections, no sticky sessions.
- A redirect is I/O-bound (one cache lookup + a tiny response), so one instance realistically does ~2K–5K req/s.

```
Sizing:
  4,000 req/s reaching origin ÷ ~3,000 req/s per instance ≈ 2 instances
  Plan for the no-CDN worst case (10,000 ÷ 3,000 ≈ 4), 2× spikes and N+1 failure → 8–10 instances
  Autoscale on CPU / req-per-instance / p99 latency (e.g. target 50% CPU)
```

- **The ~3K req/s per instance is an assumption, not a benchmark; validate it with a load test.** A redirect is mostly waiting on I/O (one Redis lookup, ~5 ms end to end), so by Little's Law 3K req/s needs only ~15 requests in flight, which is easy for an async server. 3K is deliberately conservative to leave room for TLS, logging, metrics and GC pauses. Thread-per-request stacks (e.g. classic Tomcat) may only reach 500–1.5K req/s, so re-size with `instances = peak req/s ÷ measured per-instance capacity, plus headroom`.
- Non-blocking I/O and keep-alive/HTTP/2 between LB and service; deploy across multiple AZs so losing one keeps capacity.

#### Supporting choices (keep the core path fast)

- **Analytics is async:** publish a click event to Kafka and return. 10K events/s is small for Kafka; batch sends (`linger.ms` 5–20 ms, compression) and use a bounded buffer so a Kafka outage drops events rather than failing redirects. No counters, joins or writes in the redirect request.
- **Regional deployment:** Read Service + Redis + DB replica per region with GeoDNS/anycast. This divides the 10K/s across regions and cuts latency; writes still go to one primary region (see [Global Distribution](#17-global-distribution--geo-routing)).

### Optional hardening (for unusual cases, not the foundation)

The design above already handles 10K/s. These add protection against specific edge cases; mention them **after** the core path.

| Technique | Problem it addresses | How it helps |
|---|---|---|
| **Single-flight / request coalescing** | A hot key expires in Redis and 100 concurrent requests all miss | One DB read per key per instance; the other 99 wait for the result instead of causing 99 more reads |
| **TTL jitter / stale-while-revalidate** | Many keys expiring together | Spreads refreshes; serves stale data while refreshing |
| **Bloom filter of valid codes** | Bots request random non-existent codes, which always miss and reach the DB | Rejects likely-nonexistent codes before the DB (no false negatives; rare false positives pass through to the DB) |
| **Negative caching (30–60 s)** | Repeated lookups of the same missing/expired code | Caches the 404/410 so the DB isn't re-queried |
| **In-process (L1) cache, 5–60 s TTL** | One viral link concentrates traffic on a single Redis shard | Serves the hottest keys from memory, removing the network hop and shard hot spot |
| **Cache warming** | Cold Redis node (restart/flush) means 0% hit rate | Preload top-N hot keys; roll restarts gradually |
| **Load shedding / DB concurrency cap** | Redis outage sends everything to the DB | Cap concurrent DB queries; serve stale or return 503 rather than queueing |

### Beyond 10K

| Target | What changes |
|---|---|
| 10K → 100K req/s | Add Read Service instances and Redis shards; raise CDN TTLs for permanent links. Architecture is unchanged. |
| 100K → 1M req/s | Multi-region active-active reads; more Redis shards; the CDN does most of the work; shard the DB; consider edge KV/workers. |
| One viral link (1M req/s on one code) | CDN absorbs it (one cached object); if Redis is hit, replicate the hot key across shards (`key#1..key#N`) or add an L1 cache. |

### Validate and monitor

- **Load test** to 2× target (20K req/s) using a realistic Zipf (power-law) key distribution plus some misses/404s; uniform keys make the cache look better than it is.
- **Watch:** hit rate per layer (CDN, Redis), p50/p99 latency, DB QPS and connections, Redis ops/s and hot keys, Kafka producer errors.
- **Alert** when Redis hit rate drops below ~90%, DB QPS spikes, or p99 exceeds 100 ms.

### 60–90 second talk track

> "10K redirects a second is about 9× our average, but it's a friendly workload: read-only, key-value, with a tiny hot set of about 250 MB. So I scale by absorbing traffic in three layers, so the database never sees 10K.
>
> First, the CDN caches redirects at the edge. Permanent links get long TTLs, editable ones get short TTLs with a purge on change, so viral spikes and repeat clicks mostly never reach origin. Say 60% hit, so about 4K a second gets through.
>
> Second, the Read Service is stateless, so I scale it horizontally behind a load balancer across availability zones. At roughly 3K requests a second per instance, a handful of instances covers it, and I'd run 8 to 10 for spikes and failure, autoscaling on CPU and latency.
>
> Third, Redis handles what the CDN misses. One node does about 100K ops a second, so this fits easily, and I'd run a cluster with replicas for availability. With a 95%-plus hit rate, the database sees only a couple of hundred point reads a second, which a couple of read replicas handle. If data grows, I shard by hash of short code.
>
> Click tracking is asynchronous through Kafka so it never touches the redirect path, and I'd deploy regionally with geo routing to split load and cut latency.
>
> On top of that I'd add hardening for edge cases: single-flight so an expired hot key causes one DB read instead of a hundred, a Bloom filter and negative caching so bots probing random codes can't punch through to the database, and a small local cache if one viral link concentrates on a single Redis shard.
>
> To validate, I'd load test at 2× with a Zipf key distribution and watch hit rate per layer, p99 and DB QPS. Going to 100K is the same architecture with more instances and shards."

---

*Guide compiled for Senior & Staff-level system design interview preparation.*
*Topics: URL Shortener, Distributed Systems, Caching, ID Generation, Collision Handling, CAP Theorem.*
