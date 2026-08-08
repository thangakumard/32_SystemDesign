# Web Crawler — System Design Interview Guide
### Senior & Staff Engineer Level

---

## Table of Contents

1. [Problem Statement & Scope](#1-problem-statement--scope)
2. [Requirements](#2-requirements)
3. [Capacity Estimation](#3-capacity-estimation)
4. [High-Level Architecture](#4-high-level-architecture)
5. [Component 1: Seed URLs](#5-component-1-seed-urls)
6. [Component 2: URL Frontier](#6-component-2-url-frontier)
7. [Component 3: URL Router](#7-component-3-url-router)
8. [Component 4: Fetcher Fleet](#8-component-4-fetcher-fleet)
9. [Component 5: HTML Parser & Link Extractor](#9-component-5-html-parser--link-extractor)
10. [Component 6: Deduplication Service](#10-component-6-deduplication-service)
11. [Component 7: Storage Layer](#11-component-7-storage-layer)
12. [Component 8: DNS Cache](#12-component-8-dns-cache)
13. [Component 9: robots.txt Cache](#13-component-9-robotstxt-cache)
14. [CAP Theorem Positioning](#14-cap-theorem-positioning)
15. [Failure Modes & Mitigations](#15-failure-modes--mitigations)
16. [Scalability & Sharding](#16-scalability--sharding)
17. [Freshness & Recrawl Strategy](#17-freshness--recrawl-strategy)
18. [Senior vs Staff Answer Differentiators](#18-senior-vs-staff-answer-differentiators)
19. [Interview Time Allocation](#19-interview-time-allocation)
20. [Quick-Reference Cheatsheet](#20-quick-reference-cheatsheet)

---

## 1. Problem Statement & Scope

A **web crawler** (also called a spider or bot) is a distributed system that systematically discovers, downloads, and indexes web pages. It starts from a set of seed URLs, follows hyperlinks recursively, and stores the retrieved content for downstream consumers such as search engines, data pipelines, or ML training datasets.

### What the interviewer is really testing

- Can you design a **fault-tolerant distributed system** at internet scale?
- Do you understand **politeness, deduplication, and freshness** as first-class concerns?
- Can you proactively surface **failure modes and tradeoffs** without being prompted?
- Do you make **explicit CAP theorem decisions** per component?

---

## 2. Requirements

### Functional Requirements

| Requirement | Notes |
|---|---|
| Start from seed URLs and discover new pages | Bootstrap problem — seeds matter |
| Follow hyperlinks recursively | Build and traverse the web graph |
| Respect `robots.txt` | Legal and ethical requirement |
| Deduplicate URLs and content | Avoid redundant crawls |
| Store raw HTML and extracted metadata | For downstream indexing |
| Support recrawling for freshness | Pages change over time |
| Handle redirects | Up to N hops; detect cycles |

### Non-Functional Requirements

| Requirement | Target |
|---|---|
| Scale | ~1 billion pages/day |
| Politeness | ≥ 1–5s delay between requests to same domain |
| Fault tolerance | Worker crashes must not lose URLs |
| Extensibility | Plug-in parsers, storage backends |
| Freshness | News sites recrawled in hours; static pages in weeks |

---

## 3. Capacity Estimation

```
Target: 1 billion pages/day

Throughput:
  1,000,000,000 / 86,400 ≈ 11,574 pages/second

Bandwidth (avg 100KB/page):
  11,574 × 100KB ≈ 1.16 GB/s raw fetch throughput

Concurrency (avg 500ms fetch time):
  11,574 × 0.5s ≈ 5,787 concurrent connections

Fetcher nodes (500 connections/node):
  5,787 / 500 ≈ 12 nodes  →  use 36 nodes (3× headroom)

Bloom filter for URL dedup (1B URLs, 1% FP rate):
  ~10 bits/URL × 1B = 1.25 GB  →  fits in Redis

Storage (100KB HTML × 1B pages):
  ≈ 100 TB/day raw HTML  →  object store (S3/GCS)

Kafka throughput:
  1.16 GB/s  →  ~12 partitions at 100MB/s each
```

**Key insight:** Storage is the dominant cost, not compute. Raw HTML at 100TB/day means you need a tiered strategy — hot recent content on fast storage, older content on cheaper object storage.

---

## 4. High-Level Architecture

```
                        ┌─────────────────┐
                        │   Seed URLs     │
                        │ (bootstrap set) │
                        └────────┬────────┘
                                 │
                    ┌────────────▼─────────────┐
                    │   URL Frontier           │
                    │   (Priority Scheduler)   │
                    │   Per-domain queues      │
                    │   Politeness heap        │
                    └────────────┬─────────────┘
                                 │
          ┌──────────────────────┼──────────────────────┐
          │                      │                      │
┌─────────▼──────┐   ┌───────────▼────────┐  ┌────────▼───────┐
│  Fetcher 1     │   │  Fetcher 2         │  │  Fetcher N     │
│  HTTP/DNS      │   │  HTTP/DNS          │  │  HTTP/DNS      │
│  robots.txt    │   │  robots.txt        │  │  robots.txt    │
└─────────┬──────┘   └───────────┬────────┘  └────────┬───────┘
          │                      │                      │
          └──────────────────────▼──────────────────────┘
                                 │  (Kafka topic: raw-pages)
                    ┌────────────▼──────────┐
                    │   HTML Parser         │
                    │   Link Extractor      │
                    │   Metadata Extractor  │
                    └──────┬────────┬───────┘
                           │        │
               ┌───────────▼──┐  ┌──▼──────────────┐
               │  Dedup       │  │  Storage Layer   │
               │  Service     │  │  Object Store    │
               │  Bloom+      │  │  URL Store       │
               │  SimHash     │  │  Search Index    │
               └───────┬──────┘  └─────────────────┘
                       │
               new URLs feed back into frontier
```

**Kafka** sits between every layer as the durable message bus. This decouples components, provides replay capability (re-parse without re-crawl), and enables back-pressure via consumer lag monitoring.

---

## 5. Component 1: Seed URLs

### What it is

The initial set of URLs injected into the frontier to bootstrap the crawl. Without seeds, the crawler has no starting point.

### Sources (in priority order)

| Source | Quality | Notes |
|---|---|---|
| Prior crawl data | ★★★★★ | Best seeds come from your own previous crawl's high-PageRank URLs |
| Web directories (DMOZ/Curlie) | ★★★★☆ | Human-curated, broad topic coverage |
| Wikipedia external links | ★★★★☆ | High quality, globally diverse |
| Sitemap.xml files | ★★★★☆ | Site-declared canonical URLs; structured by priority |
| DNS zone files (ICANN) | ★★☆☆☆ | 200M+ .com domains; very noisy — filter heavily |
| Social signals (Reddit, Twitter) | ★★★☆☆ | Good for freshness; recent content discovery |
| Common Crawl corpus | ★★★☆☆ | Free petabyte-scale dataset; inherit decades of discovery |

### Seed URL Quality Filters

Before injecting seeds into the frontier, apply:

1. **DNS resolution check** — does the domain resolve? Eliminates dead domains.
2. **HTTP reachability** — does HEAD return 2xx/3xx? Filters parked/error pages.
3. **robots.txt allowance** — is crawling permitted?
4. **Spam/blocklist check** — cross-reference Spamhaus, SURBL.
5. **Link-out density** — parse homepage; prefer seeds with 50+ diverse outbound links.
6. **Language/encoding detection** — filter by target language for scoped crawlers.

### The Seed Bias Problem (Staff-level concern)

> The topology of your crawl is largely determined within the first 2–3 hops from your seeds.

If seeds are English-centric and US-biased, your frontier fills with English-language content before it can discover the Arabic, Chinese, or Spanish web. This is a **structural bias** that compounds over time.

**Mitigations:**
- **Stratified seeding** — explicitly allocate seed budget by TLD/language (N seeds per `.de`, `.jp`, `.br`, etc.)
- **Recurrent seed injection** — continuously inject fresh seeds from sitemaps and DNS zone deltas; don't seed once at startup
- **Frontier diversity enforcement** — scheduler monitors domain/language distribution and suppresses over-represented clusters

### What NOT to discuss here

- DFS vs BFS — that belongs in frontier traversal strategy, not seed selection
- Specific ranking algorithms — seeds don't need algorithmic depth; the interesting work is in the frontier scheduler

**Time to spend in interview:** ~3 minutes. Mention sources, call out the seed bias problem, pivot to the frontier.

---

## 6. Component 2: URL Frontier

The URL frontier is **the heart of the crawler**. It is simultaneously a priority queue, a rate limiter, a scheduler, and a politeness enforcer.

### The Two-Tier Design

```
TIER 1: Front Queues (by priority)          TIER 2: Back Queues (by domain)

┌─────────────────────┐                    ┌──────────────────────────┐
│  P1 — High          │──┐                 │  example.com  [FIFO]     │
│  (news, homepages)  │  │  Prioritizer    │  url1 → url2 → url3      │
├─────────────────────┤  │  (weighted  ──▶ ├──────────────────────────┤
│  P2 — Medium        │──┤   round-        │  news.site    [FIFO]     │
│  (category pages)   │  │   robin)        │  url4 → url5 → ...       │
├─────────────────────┤  │                 ├──────────────────────────┤
│  P3 — Low           │──┘                 │  shop.io      [FIFO]     │
│  (deep archives)    │                    │  url6 → ...              │
└─────────────────────┘                    └──────────────────────────┘
                                                        │
                                           ┌────────────▼─────────────┐
                                           │   Domain Heap            │
                                           │   Min-heap by            │
                                           │   next_ok_timestamp      │
                                           └────────────┬─────────────┘
                                                        │
                                           ┌────────────▼─────────────┐
                                           │   Fetcher Fleet          │
                                           │   Polls heap · dequeues  │
                                           │   top-eligible domain    │
                                           └──────────────────────────┘
```

### Front Queues — Priority Scoring

URLs are scored on arrival and bucketed into priority queues. Scoring inputs:

| Signal | Weight | Notes |
|---|---|---|
| PageRank / inbound link count | High | Pre-computed from prior crawl |
| Domain authority | High | Alexa rank, domain age, TLD type |
| Content freshness signal | High | News domains score higher |
| Link depth from seed | Medium | Shallower = higher priority |
| Sitemap `<priority>` tag | Medium | Explicit site preference |

**Weighted round-robin** between queues: P1 picked 3× more often than P2; P2 picked 3× more often than P3.

### Back Queues — Per-Domain Politeness

Each domain gets exactly **one FIFO back queue**. This is the critical design that makes politeness work at scale.

```
Domain entry = {
  domain:        "example.com",
  queue:         [url1, url2, url3, ...],   // FIFO
  last_crawl_ts: 1718000000,
  crawl_delay:   2,                          // seconds (from robots.txt or default)
  next_ok_ts:    1718000002                  // last_crawl + crawl_delay
}
```

### Domain Heap — The Scheduling Engine

A **min-heap keyed on `next_ok_ts`**. The fetcher always peeks at the root — the domain with the earliest eligibility time:

```
Scheduling loop:
  1. Peek top of heap → domain D with smallest next_ok_ts
  2. If now < next_ok_ts → sleep until next_ok_ts
  3. Dequeue next URL from D's back queue
  4. Check robots.txt cache → discard if path disallowed
  5. Hand URL to fetcher
  6. On fetch complete → reinsert D with new next_ok_ts = now + crawl_delay
```

**Why a heap?** O(log N) insert and peek operations. At 100K domains in the active frontier, this is negligible overhead.

### Politeness Delay Calculation

`crawl_delay` comes from three sources in priority order:

1. `Crawl-delay` directive in `robots.txt` (must be respected)
2. `Request-rate` directive (extended `robots.txt` spec, rare)
3. Your configured default (1–5 seconds for a well-behaved crawler)

**Critical nuance:** Delay should be measured from **end of previous fetch**, not start. A 3-second download + 2-second delay = the server sees a request every 5 seconds, not every 2. Getting this wrong inadvertently hammers slow servers.

### Frontier Storage (Two-Tier)

| Tier | System | Contents | Why |
|---|---|---|---|
| Hot | Redis | Domains with `next_ok_ts` in next 5–15 min | Sub-ms reads for active scheduling |
| Cold | Cassandra | Full frontier — billions of URLs awaiting recrawl | Wide-row model, partition by domain hash |

A background job continuously **promotes** domains from cold → hot as their `next_ok_ts` approaches.

### Staff-Level Gotchas

**Domain starvation problem:** If front queues heavily favour high-priority URLs, newly discovered low-authority domains may sit in back queues indefinitely. Fix: impose a **minimum crawl guarantee** — every enqueued domain gets at least one crawl within N days regardless of priority score.

**Coordinator bottleneck:** A single frontier coordinator becomes a hot spot above ~10K fetchers. Fix: **shard the frontier by domain hash** — shard 0 owns all domains where `hash(domain) % N == 0`. Each shard has its own heap, coordinator, and fetcher pool.

**New domain race condition:** Two fetchers simultaneously discover URLs from the same new domain and both race to create a back queue. Fix: use Redis `SET NX` (set-if-not-exists) as a distributed lock for queue creation, or use CAS (compare-and-swap) semantics on the domain map.

---

## 7. Component 3: URL Router

The URL Router sits between the prioritizer and the per-domain back queues. Its job: given a URL, determine which back queue it belongs to.

### Processing Pipeline

```
URL from prioritizer
        │
        ▼
┌───────────────────────────────────────────┐
│  Step 1: Normalize URL                    │
│  • lowercase scheme + host                │
│  • remove default ports (:80, :443)       │
│  • resolve path segments (/a/b/../c→/a/c) │
│  • sort query parameters                  │
│  • strip tracking params (utm_*, fbclid)  │
│  • strip fragment (#section)              │
│  • canonicalize trailing slash            │
└───────────────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────────────┐
│  Step 2: Extract Routing Key              │
│  • full hostname for most domains         │
│  • eTLD+1 for shared-infra hosts          │
│    (user1.github.io → github.io)          │
│  • IP literal for IP URLs                 │
└───────────────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────────────┐
│  Step 3: Bloom Filter Dedup Check         │
│  • URL already seen? → DROP               │
│  • No? → mark seen · continue             │
└───────────────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────────────┐
│  Step 4: Hash to Back Queue               │
│  queue_id = hash(domain) % num_queues     │
│  • Fixed assignment preserves FIFO order  │
│  • Consistent hashing for elastic scaling │
└───────────────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────────────┐
│  Step 5: Insert                           │
│  • New domain? Create queue + heap entry  │
│  • Existing domain? Append to tail        │
└───────────────────────────────────────────┘
```

### Why Route by Domain, Not URL Hash?

A natural question: why not `hash(url) % N` for perfectly even distribution?

Because **politeness requires domain locality**. To enforce a crawl delay for `example.com`, you need to know when it was last fetched. If two URLs from `example.com` land in different queues on different machines, you need cross-machine coordination on every fetch — turning a local O(1) check into a distributed lock on the hot path.

Domain routing keeps all URLs for a domain in one place → politeness tracking is local state → no coordination overhead.

**Tradeoff:** Uneven queue sizes (a hot domain like `reddit.com` may have millions of pending URLs). Fix: detect hot domains (queue depth > threshold) and **split into sub-queues** by URL hash within the domain. Each sub-queue still respects the same per-domain crawl delay.

### eTLD+1 and the Shared Infrastructure Problem

`blog.example.com` and `shop.example.com` → different servers → separate queues. Route on full hostname.

`user1.github.io` and `user2.github.io` → same origin server → rate-limit collectively. Route on eTLD+1 (`github.io`). Requires a **Public Suffix List (Mozilla PSL)** lookup per domain.

---

## 8. Component 4: Fetcher Fleet

Each fetcher is a **stateless worker** responsible for making HTTP requests and delivering raw content to the parser.

### Fetch Pipeline per Worker

```
1. Pull URL batch from assigned domain queue (via heap scheduler)
2. Check robots.txt cache → abort if path disallowed
3. Resolve DNS (from local TTL-aware cache)
4. Open TCP connection + TLS handshake
5. Send HTTP GET with proper User-Agent and headers
6. Handle redirects (up to N hops; track canonical URL)
7. Stream response body; enforce size limit (e.g. 5MB max)
8. Push raw HTML + metadata to Kafka topic: raw-pages
9. Update domain's last_crawl_ts in frontier
```

### HTTP Headers to Set

```http
User-Agent: MyCrawler/1.0 (+https://example.com/crawler-info)
Accept: text/html,application/xhtml+xml
Accept-Language: *
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
```

Always identify your crawler honestly. Deceptive User-Agent strings violate ToS and damage trust.

### Redirect Handling

```
Max redirect chain: 5 hops
Track: [original_url → redirect_1 → redirect_2 → ... → final_url]

On each redirect:
  • Check if final_url is already in dedup filter → stop if seen
  • Check robots.txt for the new domain (may differ from original)
  • Detect cycles: if any URL in chain already visited → abort
```

Store the **canonical URL** (final destination after all redirects) as the primary key in storage.

### Spider Trap Detection

Spider traps are server-generated infinite URL spaces that can fill your frontier:

```
Examples of traps:
  /calendar/2024/01/01/
  /calendar/2024/01/02/
  /calendar/2024/01/03/  ... (infinite)

  /search?page=1&sort=asc&filter=new
  /search?page=2&sort=asc&filter=new  ... (combinatorial)
```

**Mitigations:**
- **Max path depth cap** — don't crawl URLs deeper than N path segments (e.g. 10)
- **Query parameter count limit** — discard URLs with > M unique parameters
- **Duplicate content detection** — SimHash catches pages that are parametric variants of each other
- **URL pattern detection** — if 1000+ URLs from a domain match a regex pattern, flag and throttle

### JavaScript-Rendered Pages

A critical architectural split at Staff level:

```
Standard HTML pages (majority):
  → Regular fetcher fleet (cheap, fast)

JavaScript-heavy pages (SPA frameworks):
  → Separate headless browser fleet
  → Playwright or Puppeteer workers
  → 10–50× more expensive per page
  → Separate queue, separate budget
```

**Detection heuristics for JS-rendered pages:**
- `<noscript>` block contains meaningful content
- Response HTML body < 2KB but page has significant outbound links (pre-render)
- Known JS-heavy domain list (built from prior crawls)

**Senior-level answer:** Mention JS rendering as a problem.
**Staff-level answer:** Describe it as a **separate fleet with its own queue, cost model, and capacity planning** — not an afterthought bolted onto the main fetchers.

### Fetcher Failure Handling

| Failure | HTTP Code | Action |
|---|---|---|
| Temporary error | 429, 503, 504 | Exponential backoff; requeue with delay |
| Permanent error | 404, 410 | Mark URL as dead; don't requeue |
| Redirect loop | — | Abort after N hops; mark as trap |
| Timeout | — | Retry up to 3×; then mark as failed |
| Content too large | — | Truncate or skip; log |
| TLS error | — | Retry once; then skip |

---

## 9. Component 5: HTML Parser & Link Extractor

The parser consumes raw HTML from Kafka and extracts:

### What to Extract

```
From each page:
  1. Outbound links         → feed back into URL frontier
  2. Canonical URL          → <link rel="canonical" href="...">
  3. Page title             → <title> tag
  4. Meta description       → <meta name="description">
  5. Robots meta directive  → <meta name="robots" content="noindex,nofollow">
  6. Structured data        → JSON-LD, OpenGraph, Schema.org
  7. Language               → <html lang="..."> or Content-Language header
  8. Content body text      → for indexing and SimHash computation
  9. Last-modified header   → for freshness scheduling
```

### Link Normalization (Parser's Responsibility)

All extracted links must be normalized before feeding back to the frontier:

```
Relative URL resolution:
  Base:     https://example.com/blog/post-1
  Link:     ../images/photo.jpg
  Resolved: https://example.com/images/photo.jpg

Fragment stripping:
  https://example.com/page#section → https://example.com/page

Protocol-relative URLs:
  //cdn.example.com/script.js → https://cdn.example.com/script.js
```

### Respecting Parser-Level Exclusions

Before feeding a link back to the frontier, check:

1. **`<meta name="robots" content="nofollow">`** — don't follow any links on this page
2. **`<a rel="nofollow" href="...">`** — don't follow this specific link
3. **`<meta name="robots" content="noindex">`** — don't store this page's content
4. **Link points to excluded file type** — skip `.pdf`, `.zip`, `.exe` etc. unless specifically crawling those

### Parser Architecture

Parsing is CPU-intensive. Use a **separate parser fleet** consuming from Kafka:

```
Kafka topic: raw-pages
    │
    ▼
Parser workers (stateless, horizontally scalable)
    │
    ├──▶ Kafka topic: extracted-links  →  feeds URL frontier
    └──▶ Kafka topic: parsed-content   →  feeds storage layer
```

Decouple parsing from fetching so you can:
- **Re-parse** historical HTML without re-crawling
- **Scale independently** (parsing is CPU-bound; fetching is I/O-bound)
- **Change parsing logic** and replay from stored raw HTML

---

## 10. Component 6: Deduplication Service

Dedup operates at **two independent layers**. Missing either one is a Senior-level gap; understanding both is a Staff-level signal.

### Layer 1: URL-Level Dedup (Have we seen this URL before?)

**Data structure: Bloom Filter**

```
Properties:
  • Space-efficient probabilistic set membership structure
  • False positives possible (URL incorrectly marked as seen)
  • False negatives impossible (seen URL never missed)
  • No delete support (standard Bloom filter)

Sizing for 1B URLs at 1% false-positive rate:
  Bits per URL = -log2(FP_rate)² / ln(2) ≈ 9.6 bits
  Total memory = 1B × 9.6 bits ≈ 1.2 GB  →  fits in Redis

Optimal hash functions: k = -log2(FP_rate) / ln(2) ≈ 7
```

**Bloom filter gotchas:**
- **No deletes** — for recrawl scheduling, use a **counting Bloom filter** or **TTL-keyed Redis SET** instead, which allows expiry
- **Fill ratio monitoring** — as the filter fills, false-positive rate rises. Monitor fill % and rotate to a fresh filter with a backfill window when FP rate exceeds threshold (typically 2%)
- **Distributed Bloom filter** — at scale, partition the URL space across multiple Redis nodes; each fetcher routes to the correct partition by `hash(url) % num_partitions`

**URL normalization before Bloom check** (same as router):
```
Strip: utm_* params, session IDs, trailing slashes (consistently)
Sort:  query parameters alphabetically
Lower: scheme and host
```

### Layer 2: Content-Level Dedup (Is this page a duplicate of one we've already stored?)

URL dedup alone is insufficient — the same content can appear at many different URLs (mirrors, syndication, URL parameter variants).

#### Exact Duplicate Detection

```
Algorithm: MD5 or SHA-256 hash of page body text
Storage:   Hash → URL mapping in Cassandra

If hash exists in store:
  → Skip storage
  → Still follow outbound links (the page is valid; its content is dup)
  → Log as duplicate for analytics
```

#### Near-Duplicate Detection (SimHash)

SimHash detects pages that are **mostly the same** with minor variations (different ads, timestamps, boilerplate). This is crucial for:
- Paginated listings (`/products?page=1` vs `/products?page=2`)
- Printer-friendly versions of pages
- Mobile vs desktop versions
- Regional variants with minor content changes

```
SimHash algorithm:
  1. Tokenize page text into shingles (3-word overlapping windows)
  2. For each shingle: compute hash → get 64-bit vector
  3. For each bit position i across all shingles:
       if hash_bit[i] == 1: weight[i] += shingle_weight
       if hash_bit[i] == 0: weight[i] -= shingle_weight
  4. Final SimHash: bit[i] = 1 if weight[i] > 0, else 0

Near-duplicate threshold:
  Hamming distance ≤ 3 → pages are near-duplicates
  (Hamming distance = number of differing bits in 64-bit hashes)
```

**Storage for SimHash:**
```
Store: {simhash_64bit → [url, crawl_timestamp]}

Lookup: For a new page's SimHash S, check all stored hashes
        within Hamming distance 3.

Efficient lookup trick: Split 64-bit hash into 4 × 16-bit chunks.
  Any two hashes within Hamming distance 3 must agree on at least
  one chunk. Run 4 index lookups (one per chunk), then verify
  Hamming distance on candidates.
  This reduces O(N) linear scan to O(√N) per lookup.
```

### Dedup Decision Matrix

| URL seen? | Content dup? | Action |
|---|---|---|
| Yes | — | Drop immediately at router; never fetch |
| No | Exact dup | Fetch; skip storage; follow links |
| No | Near dup | Fetch; store diff only; follow links |
| No | Unique | Fetch; store; follow links; index |

---

## 11. Component 7: Storage Layer

### Multi-Tier Storage Design

| Layer | What | System | Rationale |
|---|---|---|---|
| Raw HTML | Full page bytes | S3 / GCS | Cheap, durable; enables re-parse without re-crawl |
| URL metadata | URL, crawl time, HTTP status, depth, redirects | Cassandra | Wide-row model; partition by domain hash; high write throughput |
| Parsed content | Title, body text, anchor text, links | Elasticsearch | Full-text search + aggregations for downstream indexing |
| Frontier state | Queue pointers, domain schedules | Redis + Kafka | Sub-ms reads; durable offsets |
| Dedup hashes | URL fingerprints, SimHash values | Redis Bloom / Cassandra | Extremely high write throughput required |
| Crawl analytics | Error rates, throughput, domain health | ClickHouse / BigQuery | Time-series analytics; batch queries |

### Raw HTML Storage Schema (S3)

```
Bucket structure:
  s3://crawl-data/
    raw-html/
      year=2024/month=06/day=15/
        {crawl_id}/{domain_hash}/{url_hash}.html.gz

Metadata sidecar (JSON):
  {
    "url":           "https://example.com/page",
    "canonical_url": "https://example.com/page",
    "crawled_at":    "2024-06-15T10:30:00Z",
    "http_status":   200,
    "content_type":  "text/html; charset=utf-8",
    "content_length": 48291,
    "etag":          "\"abc123\"",
    "last_modified": "2024-06-14T08:00:00Z",
    "crawl_depth":   3,
    "referring_url": "https://example.com/"
  }
```

### URL Metadata Store (Cassandra)

```sql
CREATE TABLE url_metadata (
  domain          TEXT,
  url_hash        TEXT,
  url             TEXT,
  first_crawled   TIMESTAMP,
  last_crawled    TIMESTAMP,
  next_crawl      TIMESTAMP,
  http_status     INT,
  content_hash    TEXT,
  simhash         BIGINT,
  crawl_depth     INT,
  inbound_links   INT,
  is_duplicate    BOOLEAN,
  PRIMARY KEY (domain, url_hash)
) WITH CLUSTERING ORDER BY (url_hash ASC);
```

Partitioned by `domain` → all URLs for a domain collocated → efficient domain-level queries (e.g., "give me all crawled URLs for example.com").

### Retention Policy

```
Raw HTML:     90 days hot (S3 Standard) → then Glacier/Coldline
URL metadata: Indefinite (drives recrawl scheduling)
Parsed index: Rolling 2 years (Elasticsearch ILM policy)
Crawl logs:   30 days (operational) → 1 year (compliance)
```

---

## 12. Component 8: DNS Cache

DNS is a hidden bottleneck that most Senior-level candidates miss.

### Why It Matters

At 11,574 fetches/second, if every fetch triggers a DNS lookup:
- Each DNS lookup takes 10–100ms
- DNS resolvers have rate limits and query quotas
- DNS failures stall fetchers and reduce throughput

### DNS Cache Design

```
Structure: LRU cache keyed on hostname
           Value: {ip_address, ttl, resolved_at}

TTL handling:
  • Respect DNS TTL exactly (don't cache longer than the record says)
  • Negative caching: cache NXDOMAIN responses for 60s
    (prevents storms of failed lookups for dead domains)
  • Jitter TTL refresh: don't let all cache entries expire simultaneously

Per-fetcher cache:
  • Each fetcher node maintains its own local DNS cache
  • Avoids network round-trips to a shared DNS cache service
  • Size: ~1M entries per node (typical active domain set)

Fallback:
  • Local cache miss → query configured DNS resolver
  • Resolver miss → recursive DNS resolution
  • Timeout: 5s with 2 retries
```

### IPv6 / Dual-Stack

Implement the **Happy Eyeballs algorithm** (RFC 8305):
- Initiate IPv6 connection
- If no response within 250ms, simultaneously initiate IPv4 connection
- Use whichever responds first
- Cache the winning address family per domain

---

## 13. Component 9: robots.txt Cache

### What robots.txt Controls

```
Sample robots.txt:
  User-agent: *
  Disallow: /admin/
  Disallow: /private/
  Crawl-delay: 2
  Sitemap: https://example.com/sitemap.xml

  User-agent: MyCrawler
  Allow: /public/
  Disallow: /
```

### Cache Design

```
Key:   hostname (e.g., "example.com")
Value: {
  rules:        parsed allow/disallow rules per user-agent,
  crawl_delay:  integer seconds (from Crawl-delay directive),
  sitemap_urls: [list of sitemap URLs to fetch],
  fetched_at:   timestamp,
  ttl:          3600  (1 hour default; up to 24h for stable sites)
}

Cache storage: Redis hash per domain
Cache miss:    Fetch /robots.txt → parse → store → return rules
Cache hit:     Return stored rules directly

On robots.txt fetch failure:
  • 404 → assume everything allowed; cache this assumption for TTL
  • 5xx → retry with backoff; on persistent failure: assume allowed
  • Timeout → assume allowed; retry on next crawl cycle
```

### Matching Algorithm

```
For a given URL path and user-agent:
  1. Find most specific User-agent match (exact > wildcard *)
  2. Find longest matching Allow rule
  3. Find longest matching Disallow rule
  4. If Allow path length ≥ Disallow path length → allowed
  5. If no rules match → allowed

Path matching is prefix-based:
  Disallow: /private/
  Matches:  /private/, /private/page, /private/a/b/c
  No match: /privateer/, /public/private/
```

### Staff-Level Note

`robots.txt` is a **convention, not a technical enforcement mechanism**. Honoring it is:
1. An ethical obligation (your crawler is a guest on their server)
2. A legal consideration (Computer Fraud and Abuse Act in the US; various equivalents elsewhere)
3. A practical concern (violating it gets your IPs blocked)

Always re-fetch `robots.txt` on HTTP 5xx from the target domain — server configuration may have changed.

---

## 14. CAP Theorem Positioning

At Staff level, you must explicitly address CAP for each component. Here is the full breakdown:

| Component | CAP Choice | Reasoning |
|---|---|---|
| URL Frontier | **AP** | Better to crawl a URL twice than lose crawl progress. Eventual consistency is fine. |
| Dedup store (Bloom filter) | **CP** | False negatives (missing a dup) are far cheaper than false positives (skipping unique content). Under partition, prefer consistency: pause rather than risk incorrect dedup. |
| Raw HTML store (S3) | **AP** | Content is append-only; writes never conflict. Availability over consistency. |
| URL metadata (Cassandra) | **AP** | Tunable consistency. Use `QUORUM` for reads when freshness matters; `ONE` for high-throughput writes. |
| robots.txt cache (Redis) | **AP** | Stale robots.txt is acceptable for short TTL windows; availability (keep crawling) preferred. |
| DNS cache | **AP** | Stale DNS entry is better than a stall. Short TTLs bound the staleness window. |
| Crawl coordination (ZooKeeper/etcd) | **CP** | Coordinator decisions (shard assignments, leader election) must be consistent. Availability sacrificed during partition. |

---

## 15. Failure Modes & Mitigations

| Failure | Symptom | Detection | Mitigation |
|---|---|---|---|
| Fetcher crash mid-batch | URLs in-flight are lost | Kafka consumer lag drops; offset not committed | At-least-once delivery via Kafka; commit offset only after successful push to parser queue |
| Spider trap | Frontier fills with URL variants from one domain | Domain queue depth > threshold | URL normalization; path depth cap; query param count limit; pattern detection |
| Hot domain monopoly | One domain starves all others | Frontier imbalance metrics | Per-domain queues; politeness delays; minimum crawl guarantee for other domains |
| Bloom filter saturation | False-positive rate rises; valid URLs skipped | Monitor fill ratio | Alert at 70% fill; rotate to fresh filter with backfill window |
| DNS storm | All fetchers stall on cache miss | DNS error rate spike | Local per-fetcher DNS cache; negative caching; circuit breaker |
| Stale robots.txt | Crawl rule changes missed | 403 errors increase | Cache with short TTL (1–24h); re-fetch on 5xx |
| Clock skew between nodes | Freshness scheduling incorrect | Recrawl intervals drift | Use logical clocks (Lamport timestamps) for recrawl scheduling; NTP on all nodes |
| JS-heavy pages return empty HTML | Index misses dynamic content | Low text extraction rate for known JS domains | Separate headless browser fleet; detect via `<noscript>` pattern |
| IP ban by target site | 403/429 errors from domain | Error rate per domain | Respect robots.txt and crawl-delay; IP rotation for legitimate crawlers; reduce rate |
| Kafka partition lag | Parser falls behind fetchers | Consumer lag monitoring | Scale parser fleet; increase Kafka partitions; apply back-pressure to fetchers |
| Redirect bomb | Fetcher stuck following infinite redirects | High redirect depth counter | Hard limit: abort chain after 5 hops; detect cycles in redirect chain |
| Large page stalls fetcher | 500MB HTML page blocks worker | Response size monitoring | Stream response; enforce 5MB size limit; skip oversized pages |

---

## 16. Scalability & Sharding

### Frontier Sharding

```
Shard assignment:
  shard_id = hash(domain) % num_shards

Each shard owns:
  • Its slice of the domain namespace
  • Its own min-heap
  • Its own back queue set
  • Its own fetcher pool

Kafka routing:
  Produce to partition = hash(domain) % num_partitions
  → naturally co-locates all domain URLs to one partition
  → one fetcher pool per partition = domain locality preserved
```

### Horizontal Scaling Plan

| Component | Scaling Strategy |
|---|---|
| Fetcher fleet | Add nodes; each subscribes to a Kafka partition |
| Parser fleet | Add nodes; stateless consumers on Kafka |
| Frontier coordinator | Shard by domain hash |
| Bloom filter | Partition across Redis cluster |
| Cassandra | Add nodes; consistent hash ring; replication factor 3 |
| S3/GCS | Managed; scales automatically |
| Elasticsearch | Add data nodes; increase shard count |

### Backpressure

If the parser fleet falls behind the fetcher fleet, Kafka consumer lag grows. Apply backpressure:

```
Monitor: consumer_lag = latest_offset - committed_offset

If consumer_lag > threshold:
  → Reduce fetcher throughput (increase crawl-delay defaults)
  → OR scale up parser fleet
  → Alert on-call if lag exceeds SLO
```

---

## 17. Freshness & Recrawl Strategy

### The Problem

Different pages have radically different change frequencies:
- News articles: change in minutes/hours
- E-commerce product pages: change daily
- Wikipedia articles: change weekly
- Academic papers: never change (after publication)

A fixed recrawl interval wastes resources on static content and misses changes on dynamic content.

### Approach 1: Fixed TTL (Senior-level answer)

```
Domain type → Recrawl interval
News sites  → 1 hour
E-commerce  → 24 hours
Blogs       → 7 days
Archives    → 30 days
```

Simple to implement. Wastes crawl budget on static pages and still misses some changes.

### Approach 2: Adaptive Freshness (Staff-level answer)

Model each page's change frequency as a **Poisson process**:

```
Observation: Each time we crawl a page, we check if content changed
             (by comparing content hash or Last-Modified header)

Estimation:
  λ (change rate) = number_of_changes / total_observation_time

Optimal recrawl interval:
  T = 1 / λ  (minimises expected staleness)

Update rule:
  After each crawl:
    if content changed:  increase λ (crawl more often)
    if content same:     decrease λ (crawl less often)
    use exponential moving average for λ to handle seasonality
```

This is the approach described in Cho & Garcia-Molina's foundational crawling papers and is used in practice at Google/Bing scale.

### Change Detection Signals

```
Priority order:
  1. HTTP 304 Not Modified (server confirms no change; cheapest)
  2. ETag header comparison (hash-based change detection)
  3. Last-Modified header comparison
  4. Content hash comparison (MD5 of body text)
  5. SimHash comparison (detects minor edits)
```

---

## 18. Senior vs Staff Answer Differentiators

### Senior-level expectations

- Correctly identify all major components (frontier, fetcher, parser, dedup, storage)
- Explain URL deduplication with Bloom filters
- Mention robots.txt and crawl-delay
- Describe basic priority scheduling
- Handle obvious failure modes (fetcher crash, redirect loops)
- Provide basic capacity math

### Staff-level expectations (additional)

| Topic | What Staff-level adds |
|---|---|
| Seed URLs | Proactively raise the seed bias problem and stratified seeding |
| URL Frontier | Two-tier design (front + back queues); domain heap; starvation problem; sharding |
| Dedup | Both URL-level AND content-level; SimHash algorithm with Hamming distance math; FP rate sizing |
| JS rendering | Separate headless browser fleet with distinct cost model, not an afterthought |
| DNS | Per-fetcher TTL-aware cache; negative caching; Happy Eyeballs for dual-stack |
| robots.txt | Matching algorithm detail; legal/ethical framing; failure handling |
| Storage | Explicit per-component system choices with reasoning; retention policy |
| CAP theorem | Explicit AP vs CP decision for every component with reasoning |
| Freshness | Poisson process model; adaptive recrawl intervals; ETag/304 optimization |
| Failure modes | Proactively enumerate spider traps, Bloom saturation, clock skew, IP bans |
| Politeness math | Delay from end-of-fetch, not start; crawl-delay source priority |
| Coordinator bottleneck | Frontier sharding by domain hash to remove single-coordinator bottleneck |

### The single most important Staff differentiator

**Proactively surfacing failure modes** without being asked. A Staff engineer owns the operational story of the system they design. Before the interviewer asks "what happens if a fetcher crashes?", you should already have addressed it. Before they ask "what about spider traps?", you've already called it out.

If you find yourself only answering questions the interviewer asks rather than anticipating them, you're presenting at Senior level, not Staff.

---

## 19. Interview Time Allocation

For a 45-minute system design session:

| Phase | Time | Focus |
|---|---|---|
| Requirements & scope | 5 min | Confirm functional + non-functional; 1B pages/day target |
| Capacity estimation | 5 min | Pages/sec, bandwidth, connections, storage, Bloom filter size |
| High-level architecture | 5 min | Draw all components; name the message bus (Kafka) |
| URL Frontier (deep dive) | 12 min | Two-tier design, heap, politeness, storage, gotchas |
| Deduplication | 8 min | Bloom filter math, SimHash, two-layer design |
| Fetcher fleet | 7 min | Pipeline, redirect handling, JS fleet, failure handling |
| Storage layer | 3 min | Per-component system choices with brief reasoning |
| Failure modes | 5 min | Spider traps, Bloom saturation, clock skew, IP bans |
| Seed URLs (if time) | 3 min | Sources, bias problem, stratified seeding |

**What to cut if short on time:** Storage layer detail and seed URL depth. **Never cut:** URL Frontier, Dedup, or Failure modes.

---

## 20. Quick-Reference Cheatsheet

```
KEY ALGORITHMS
──────────────
URL Dedup:        Bloom filter  |  1.2GB for 1B URLs at 1% FP rate  |  7 hash functions
Content Dedup:    SimHash       |  64-bit hash  |  Hamming distance ≤ 3 = near-duplicate
Scheduling:       Min-heap      |  keyed on next_ok_ts  |  O(log N) insert/peek
Priority:         Weighted round-robin across front queues
Recrawl:          Poisson process  |  T = 1/λ optimal interval

KEY NUMBERS
───────────
1B pages/day  →  11,574 pages/sec
100KB/page    →  1.16 GB/s bandwidth
500ms fetch   →  5,787 concurrent connections
36 fetcher nodes (with 3× headroom)
1.2 GB RAM for URL Bloom filter
100 TB/day raw HTML storage

KEY SYSTEMS
───────────
Message bus:    Kafka (replay, partition-per-domain, back-pressure)
URL frontier:   Redis (hot) + Cassandra (cold)
Raw HTML:       S3 / GCS (object store)
URL metadata:   Cassandra (partition by domain)
Search index:   Elasticsearch
Dedup hashes:   Redis Bloom filter + Cassandra
Coordination:   ZooKeeper or etcd (CP, for shard assignment)

CAP DECISIONS
─────────────
AP:  URL Frontier, Raw HTML store, URL metadata, DNS cache, robots.txt cache
CP:  Dedup store, Crawl coordination (ZooKeeper/etcd)

POLITENESS RULES
────────────────
1. Always respect robots.txt
2. Use Crawl-delay from robots.txt (default: 1–5s)
3. Measure delay from END of fetch, not start
4. One back queue per domain (not per URL)
5. Identify yourself with an honest User-Agent

STAFF-LEVEL SIGNALS TO HIT
───────────────────────────
✓ Two-layer dedup (URL + content/SimHash)
✓ JS rendering as separate fleet
✓ Seed bias problem and stratified seeding
✓ Frontier sharding to remove coordinator bottleneck
✓ Bloom filter fill-ratio monitoring and rotation
✓ Adaptive recrawl with Poisson process model
✓ Clock skew → logical timestamps for scheduling
✓ Spider trap detection and per-domain queue depth limits
✓ Explicit CAP decision per component
✓ Proactive failure mode enumeration (don't wait to be asked)
```

---

*Guide compiled for Senior & Staff-level system design interview preparation.*
*Topics: Web Crawler, Distributed Systems, URL Frontier, Deduplication, Politeness, CAP Theorem.*
