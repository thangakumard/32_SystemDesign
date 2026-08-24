# YouTube — System Design Interview Guide
### Senior & Staff Engineer Level

---

## Table of Contents

1. [Problem Statement & Scope](#1-problem-statement--scope)
2. [Requirements](#2-requirements)
3. [Capacity Estimation](#3-capacity-estimation)
4. [Core Entities](#4-core-entities)
5. [API Design](#5-api-design)
6. [High-Level Architecture](#6-high-level-architecture)
7. [Component 1: Video Upload Path](#7-component-1-video-upload-path)
8. [Component 2: Async Processing Pipeline (Chunk + Transcode)](#8-component-2-async-processing-pipeline-chunk--transcode)
9. [Component 3: Manifest Files & Streaming Protocols](#9-component-3-manifest-files--streaming-protocols)
10. [Component 4: Watch/Streaming Path & CDN](#10-component-4-watchstreaming-path--cdn)
11. [Component 5: Metadata Store](#11-component-5-metadata-store)
12. [Component 6: Adjacent Features (Comments, Views, Search, Recs)](#12-component-6-adjacent-features-comments-views-search-recs)
13. [Architectural Alternatives — Pros & Cons](#13-architectural-alternatives--pros--cons)
14. [CAP Theorem Positioning](#14-cap-theorem-positioning)
15. [Failure Modes & Mitigations](#15-failure-modes--mitigations)
16. [Scalability & Sharding](#16-scalability--sharding)
17. [Live Streaming (Extension Topic)](#17-live-streaming-extension-topic)
18. [Senior vs Staff Answer Differentiators](#18-senior-vs-staff-answer-differentiators)
19. [Interview Time Allocation](#19-interview-time-allocation)
20. [Quick-Reference Cheatsheet](#20-quick-reference-cheatsheet)

---

## 1. Problem Statement & Scope

Design **YouTube**: a video-sharing platform where users upload, store, and stream video content at massive scale. The same fundamentals (chunked upload, transcoding, adaptive bitrate streaming, CDN delivery) transfer almost directly to **Netflix, Hulu, and Spotify**-style problems — the interviewer is really testing your understanding of **large binary object handling over a network**, not video-specific trivia.

### What the interviewer is really testing

- Do you know how to move **very large files** through a system that has per-request size limits everywhere (API Gateway, load balancers, app servers)?
- Do you understand **why** you'd transcode into multiple resolutions and bitrates, not just that you should?
- Can you separate **upload-time chunking** from **playback-time chunking** — they solve different problems and are sized differently?
- Do you proactively reach for **async, event-driven processing** instead of a synchronous request/response pipeline for expensive work?
- Can you correctly apply **CAP theorem** to the upload/view split (availability over consistency)?

### Explicitly Out of Scope (call this out to your interviewer)

- Comments, likes, subscriptions, notifications — treat as extensions if time allows (see [§12](#12-component-6-adjacent-features-comments-views-search-recs))
- Recommendation/ranking algorithm internals
- Monetization/ads pipeline
- Live streaming — treated as a bonus deep dive in [§17](#17-live-streaming-extension-topic), since it changes the design significantly (no async transcoding possible, low end-to-end latency needed)

---

## 2. Requirements

### Functional Requirements

| Requirement | Notes |
|---|---|
| Users should be able to **upload** videos | Core write path |
| Users should be able to **watch / stream** videos | Core read path |

Keep this list short. YouTube's functional surface is deceptively simple — the entire interview lives in the non-functional requirements and the deep dives.

### Non-Functional Requirements

| Requirement | Target | Why it matters |
|---|---|---|
| **CAP: Availability over Consistency** | Eventual consistency on upload | A video uploaded in Germany doesn't need to be visible in the US instantly — seconds-to-minutes propagation delay is acceptable. Availability (users can always watch existing content) matters far more. |
| **Support very large files** | Up to 256 GB / 12 hours (real YouTube limit) | Breaks naive "upload the whole file in one POST" designs immediately |
| **Low-latency streaming** | First pixel in ~500 ms | Applies even in **low-bandwidth environments** (3G, bad Wi-Fi) |
| **Scalability** | ~1M uploads/day, ~100M DAU (views) | Massive read:write skew — reads dominate by orders of magnitude |

**Staff-level move:** state the CAP tradeoff explicitly and unprompted before the interviewer asks. This single sentence — *"video upload is eventually consistent, video viewing must be highly available"* — sets the frame for every design decision that follows (async pipeline, S3 durability, CDN caching).

---

## 3. Capacity Estimation

```
UPLOADS
  1,000,000 uploads/day → 1,000,000 / 86,400 ≈ 12 uploads/sec (avg)
  Peak (3-5× avg)        → ~40-60 uploads/sec

VIEWS
  100,000,000 DAU, assume 5 views/user/day avg
  = 500,000,000 views/day → ~5,787 views/sec avg, ~20-25K/sec peak

STORAGE (raw, pre-transcode)
  Assume avg video 100MB (mix of short clips and long-form)
  1M uploads/day × 100MB ≈ 100 TB/day raw ingest

STORAGE (post-transcode, multiple resolutions)
  Transcoding into ~5-6 renditions (240p...4K) roughly
  2-3× the raw storage footprint (lower resolutions are much smaller,
  higher bitrates like 4K/HDR dominate)
  ≈ 250-300 TB/day total stored (before CDN caching layer)

METADATA
  1M uploads/day × ~1KB metadata row ≈ 1 GB/day
  → trivially small; NOT a scaling bottleneck (contrast with raw video)

BANDWIDTH (egress, the real cost driver)
  500M views/day, avg watch session pulling ~50MB (mixed resolutions,
  partial views)
  ≈ 25 PB/day egress → this is why CDN caching is not optional,
  it's the single biggest cost lever in the whole system
```

**Key insight:** unlike the web crawler problem (where storage was the dominant cost), for YouTube the dominant cost is **egress bandwidth**, not storage. A single popular video watched by millions of people costs far more in repeated transfer than it does to store once. This should drive you toward CDN caching and toward *not* re-fetching from origin (S3) on every view.

---

## 4. Core Entities

Keep this lightweight at this stage — just the nouns, not full schemas yet (schemas emerge naturally once you hit the metadata store deep dive).

| Entity | Description |
|---|---|
| **Video** | The raw video bytes / binary object. Stored in blob storage, never in a relational/NoSQL row directly. |
| **VideoMetadata** | Title, description, creator, privacy setting, upload status, S3 pointers, chunk/resolution manifest. This is what actually lives in your database. |
| **User** | The uploader / viewer. Out of scope for deep auth/profile design here. |

**Why split Video from VideoMetadata explicitly:** they have completely different storage requirements (huge immutable blobs vs. small structured/queryable records) and different consistency/availability needs. Conflating them in your entity list is a signal you haven't thought about the storage layer yet — call out the split proactively.

---

## 5. API Design

Derive APIs directly from the functional requirements — one endpoint per requirement, then refine as you hit real-world limits.

### Naive first pass

```
POST /videos
  body: { videoBytes, title, description, ... }
  → uploads bytes and metadata in a single request

GET /videos/{videoId}
  → returns { videoBytes, title, description, creator, ... }
```

**Immediately call out the flaw:** most API Gateways / load balancers cap request body size (AWS API Gateway: 10MB). A 256GB video cannot go through this endpoint. This is expected to be caught by a Senior candidate with a nudge, and proactively by a Staff candidate.

### Refined APIs (post multi-part upload redesign)

```
POST /videos/metadata
  body: { title, description, sizeBytes, ... }
  → creates VideoMetadata row with status = PENDING
  → returns { videoId, presignedUploadUrls[] }   // one per chunk

PUT  <presigned S3 URL>                            // client → S3 directly
  → client uploads each chunk directly to blob storage, bypassing
    the app tier entirely

GET  /videos/{videoId}
  → returns { title, description, creator, manifestUrl, status }

GET  <manifest, via CDN or S3>
  → client fetches the manifest file (resolution → chunk URL mapping),
    then adaptively pulls video segments directly from CDN/S3
```

**The pattern to name explicitly:** the app tier should never sit in the data path for large binary payloads. It issues **pre-signed URLs** and gets out of the way; the client talks directly to blob storage (and later, CDN) for the actual bytes. This is the single most important API-design insight for this problem, and it generalizes to Dropbox, Google Drive, and any large-file system design.

---

## 6. High-Level Architecture

```
                         ┌───────────────┐
                         │    Client     │
                         └───────┬───────┘
                                 │
                       ┌─────────▼─────────┐
                       │   API Gateway     │  (auth, rate limit, routing,
                       │  (+ Load Balancer)│   NOT in the video byte path)
                       └─────────┬─────────┘
                                 │
                       ┌─────────▼─────────┐
                       │   Video Service   │  (stateless, horizontally
                       │                   │   scalable)
                       └────┬─────────┬────┘
                             │         │
                  store metadata     issue pre-signed
                             │       upload URLs
                  ┌──────────▼──────┐   │
                  │  Video Metadata │   │
                  │  DB (Postgres / │   │
                  │  DynamoDB)      │   │
                  └─────────────────┘   │
                                         │
                             ┌───────────▼────────────┐
                             │   Client uploads chunks │
                             │   directly (multipart)  │
                             └───────────┬────────────┘
                                         │
                             ┌───────────▼────────────┐
                             │   S3 / Blob Storage      │
                             │   (raw video, original)  │
                             └───────────┬────────────┘
                                         │ S3 event notification
                             ┌───────────▼────────────┐
                             │        Chunker          │  (2-10s clips,
                             │  (stateless workers)    │   key-frame aligned)
                             └───────────┬────────────┘
                                         │
                    ┌────────────────────┼────────────────────┐
                    │                    │                    │
             ┌──────▼─────┐      ┌───────▼──────┐     ┌───────▼──────┐
             │ Transcoder │      │ Transcoder   │ ... │ Transcoder   │
             │  (4K)      │      │  (1080p)     │     │  (240p)      │
             └──────┬─────┘      └───────┬──────┘     └───────┬──────┘
                    │                    │                    │
                    └────────────────────▼────────────────────┘
                             ┌───────────────────────┐
                             │  S3 (chunks per        │
                             │  resolution) + Manifest│
                             └───────────┬───────────┘
                                         │ update
                             ┌───────────▼───────────┐
                             │  Video Metadata DB     │
                             │  status = UPLOADED     │
                             └────────────────────────┘

                         ── Watch Path ──

  Client ──GET /videos/{id}──▶ Video Service ──▶ Metadata DB
     │                                          (name, desc, manifestUrl)
     │
     └──GET manifest + chunks──▶  CDN (edge, cached)  ──miss──▶ S3 (origin)
                                        │
                              adaptive bitrate fetch loop,
                              client reassesses bandwidth per chunk
```

**The message-bus role, played by S3 events instead of Kafka:** notice this design leans on **S3 event notifications** as its async trigger mechanism rather than an explicit Kafka topic (unlike the web crawler design). Both are valid — call this out to your interviewer as a deliberate choice: S3 notifications are simpler when the pipeline is a linear DAG (upload → chunk → transcode → done) with no need for replay/multiple consumer groups. If you need replay capability, multiple independent consumers, or exactly this kind of processing pipeline at 10x this scale, swapping S3 events for a **Kafka topic** (`raw-video-uploaded`) in front of the chunker is a reasonable, discussable upgrade — see [§13](#13-architectural-alternatives--pros--cons).

---

## 7. Component 1: Video Upload Path

### Why a single POST doesn't work

Every layer between client and storage has a body-size ceiling — API Gateway (10MB on AWS), load balancers, app server request buffers. A 256GB file blows through all of them by 4-5 orders of magnitude.

### The fix: Multi-part upload directly to blob storage

```
1. Client → POST /videos/metadata { title, description, sizeBytes }
2. Video Service creates VideoMetadata row, status = PENDING
3. Video Service asks S3 to initiate a multi-part upload,
   receives back N pre-signed PUT URLs (one per ~5-10MB part)
4. Video Service returns { videoId, presignedUrls[] } to client
5. Client's SDK (e.g. AWS multipart upload SDK) chunks the file
   locally and PUTs each part directly to its pre-signed URL —
   bypassing the app tier and API Gateway entirely
6. On completion, S3 stitches the parts into one object and fires
   an S3 event notification
7. A worker (Lambda / consumer) receives the notification, updates
   VideoMetadata: status = UPLOADED, s3Url = <full video location>
```

### Why not trust the client to report completion?

The client could crash, lie, or simply never call back — leaving metadata permanently `PENDING` while a valid file sits in S3, or worse, a client falsely claiming completion before all parts landed. **S3 notifications are the source of truth**, not client callbacks. This is the same trust boundary lesson as the web crawler's dedup layer: never let an untrusted actor assert system state directly.

### GCS equivalent

Google Cloud Storage has an analogous **resumable upload** API — same underlying idea (chunk client-side, upload directly to blob storage, avoid app-tier passthrough), different vendor terminology. Mentioning this signals you understand the pattern generalizes beyond AWS.

---

## 8. Component 2: Async Processing Pipeline (Chunk + Transcode)

### Two *different* kinds of chunking — don't conflate them

| | Upload chunking | Playback chunking |
|---|---|---|
| **Purpose** | Get large bytes past request-size limits reliably | Enable fast start + adaptive bitrate playback |
| **Size** | Large — 5-10MB parts | Small — 2-10 second clips |
| **Boundaries** | Arbitrary byte offsets | Aligned to video **key frames** for clean cuts |
| **Optimized for** | Upload throughput, HTTP overhead minimization | Time-to-first-pixel, mid-stream quality switching |

A common interview trap is treating these as the same chunking step. They are not — the video gets **re-chunked** after upload completes, specifically for streaming.

### Pipeline stages

```
1. Chunker (triggered by S3 "upload complete" event)
   - Splits the full video into 2-10 second segments
   - Cuts precisely at key frames (required for standalone-decodable
     segments — you can't decode a random mid-GOP byte range cleanly)
   - Stores chunks back to S3
   - Emits a "chunking complete" signal per video (or per chunk, for
     fine-grained parallelism)

2. Transcoder fleet (fan-out, one job per target resolution/bitrate)
   - Reads each chunk, re-encodes into N renditions:
     240p, 360p, 480p, 720p, 1080p, 4K (+ audio-only, HDR variants
     as needed)
   - Runs in parallel — this is the most CPU-expensive stage by far
   - Stores each resolution's chunks back to S3 under a
     resolution-partitioned key structure
   - Updates VideoMetadata's chunk manifest incrementally as each
     resolution finishes (don't block the whole video behind the
     slowest rendition — 4K may finish long after 240p)

3. Manifest generation
   - Once transcoding is sufficiently complete (or per a "minimum
     viable renditions" policy), generate/finalize the manifest file
   - Store in S3, optionally push to CDN ahead of first request
```

### What is actually inside a video file (useful grounding, rarely spoken aloud in-interview)

A video container (`.mp4`, `.mkv`) bundles: a **video codec** (compresses frames by storing key frames + inter-frame deltas/motion vectors instead of every pixel), an **audio codec**, and playback parameters (bitrate, resolution, frame rate, duration). Transcoding changes codec/bitrate/resolution to trade quality for bandwidth.

### Staff-level concern: pipeline failure handling

This DAG (chunk → transcode ×N → manifest) has multiple independent failure points. A Staff candidate proactively addresses:

- **Partial transcode failure** — if the 1080p job fails but 240p-720p succeed, don't fail the whole video; publish what's ready and retry the failed rendition independently. The manifest should support *partial* resolution availability.
- **Idempotent retries** — transcoding is expensive; a retried job must not double-charge compute or produce duplicate S3 objects. Key outputs deterministically by `(videoId, resolution, chunkIndex)`.
- **Dead-letter queue** — after N retries, route the failed job to a DLQ for manual/automated inspection rather than retrying forever or silently dropping.
- **Stuck-in-PENDING detection** — a background sweep job flags videos whose status hasn't progressed past `PENDING`/`CHUNKING` within an SLA window (e.g. 1 hour) for alerting/reprocessing.
- **Cost-aware prioritization** — transcode 240p/480p first (cheap, fast, unblocks *some* viewers quickly) before 4K (expensive, slow) — don't process resolutions in an arbitrary order.

---

## 9. Component 3: Manifest Files & Streaming Protocols

### What a manifest file is

A small JSON/XML/YAML file mapping each available resolution to its **ordered list of chunk URLs**:

```json
{
  "240p":  ["s3://.../240p/chunk_0", "s3://.../240p/chunk_1", "..."],
  "720p":  ["s3://.../720p/chunk_0", "s3://.../720p/chunk_1", "..."],
  "1080p": ["s3://.../1080p/chunk_0", "..."],
  "4k":    ["s3://.../4k/chunk_0", "..."]
}
```

This is small (well under the ~1KB/video metadata footprint estimated earlier) and cacheable — store and serve it from the CDN, not just S3.

### HLS and DASH

**HLS** (HTTP Live Streaming, Apple) and **DASH** (Dynamic Adaptive Streaming over HTTP, open standard) are the two dominant streaming protocols. Both formalize exactly the pattern above: chunk segmentation, manifest format, multi-bitrate rendition listing, and client-side adaptive logic — all running over plain HTTPS so they work transparently through existing CDNs.

**Interview framing:** you do not need to know HLS/DASH internals to pass this interview at any level, including Staff. What you must understand is the *concept* they encode — chunking, manifest-driven playback, and adaptive bitrate switching. Naming HLS/DASH is a nice-to-have flourish, not a requirement.

### Adaptive Bitrate Streaming (ABR) — the client-side loop

```
1. Client fetches the manifest (once, at playback start)
2. Client measures current network throughput
3. Client requests the next chunk at the resolution that best
   matches current bandwidth (not necessarily the same resolution
   as the previous chunk)
4. Repeat per chunk — this means a user can start on 4K at home
   Wi-Fi, walk outside onto 3G, and the client seamlessly drops to
   720p/480p mid-video without an interruption or manual action
```

This is what actually satisfies the "low-latency streaming, including in low-bandwidth environments" non-functional requirement — not just chunking alone, but chunking **plus** multi-resolution transcoding **plus** client-side adaptive selection, working together.

---

## 10. Component 4: Watch/Streaming Path & CDN

### Why direct-from-S3 doesn't scale for reads

Even after chunking, serving every chunk request from S3 directly means:
- Long round-trip time if S3's region is far from the viewer (e.g. origin in us-west, viewer in Germany)
- S3 egress cost paid on every single view of a popular video, even the millionth view of the same chunk
- No benefit from the fact that video content is extremely **cacheable** — chunks are immutable once transcoded

### CDN as the fix

```
Client → CDN edge (regional PoP) → cache hit? return chunk
                                  → cache miss? fetch from S3 (origin),
                                    cache it, then return
```

- Store **popular video chunks and manifest files** in CDN edge caches near viewers
- First viewer in a region pays the origin round-trip (cache miss); every subsequent viewer in that region gets a fast cache hit
- This directly attacks the dominant cost identified in capacity estimation: **egress bandwidth**

### Putting it together — the full watch flow

```
1. Client → GET /videos/{id}  → Video Service → Metadata DB
   returns: title, description, creator, manifestUrl
2. In parallel: Client → GET manifest (CDN, or S3 on cache miss)
3. Client picks initial resolution (based on current bandwidth
   estimate or a sane default), requests first chunk from CDN
4. Playback starts — often before step 1's metadata (title/description)
   has even finished rendering in the UI
5. Client continues fetching subsequent chunks per the ABR loop,
   adaptively switching resolution as network conditions change
```

---

## 11. Component 5: Metadata Store

### Schema (illustrative)

```sql
CREATE TABLE video_metadata (
  video_id       UUID PRIMARY KEY,
  user_id        UUID,
  title          TEXT,
  description    TEXT,
  status         TEXT,   -- PENDING | CHUNKING | TRANSCODING | UPLOADED | FAILED
  size_bytes     BIGINT,
  s3_raw_url     TEXT,
  manifest_url   TEXT,
  created_at     TIMESTAMP,
  updated_at     TIMESTAMP
);
-- GSI / secondary index on user_id → "all videos by this creator"
-- sort key on created_at if using DynamoDB, for chronological listing
```

### Choice of database

At the stated scale (1M uploads/day → ~1GB/day of metadata, trivially small), the metadata store is **not a scaling bottleneck** — this is a deliberate contrast with the raw video storage layer, and worth stating explicitly to show you've correctly identified where the actual scaling pressure lives.

- **PostgreSQL**: perfectly fine, likely fits on a single well-provisioned instance for years given the write volume. Gives you relational joins if you expand into comments/likes/subscriptions later.
- **DynamoDB**: also fine. Partition key = `video_id` for the main access pattern (fetch metadata by ID); a GSI on `user_id` supports "all of this creator's videos." Sort key on `created_at` if paired with partition key on `user_id`, to get chronological ordering per creator for free.
- **Sharding**: if this ever became a bottleneck (it realistically won't at this scale), shard by `video_id` — consistent with the web crawler guide's general principle of sharding by the entity that's queried most directly.

---

## 12. Component 6: Adjacent Features (Comments, Views, Search, Recs)

Explicitly out of scope for the core ask, but worth having a one-line answer ready if the interviewer extends the problem — this is a common Staff-level extension to probe breadth.

| Feature | One-line approach |
|---|---|
| **View count** | Don't write-through to the DB on every view (write amplification at 100M DAU). Buffer view events in a stream (Kafka), aggregate/batch-increment counters (e.g. via a counting service or approximate structure), flush periodically. Exact real-time counts are not required — eventual consistency is fine here too. |
| **Comments/Likes** | Separate tables/services (`comment_id`, `video_id`, `user_id`, `text`, `created_at`) — classic 1:N relational or wide-row model, partitioned by `video_id`. High write fan-out on popular videos is the main scaling concern; consider a dedicated comment service decoupled from the core video service. |
| **Search** | Elasticsearch/OpenSearch index over title, description, transcript (if auto-captioning exists), fed asynchronously off the same metadata pipeline — same pattern as the web crawler's parsed-content index. |
| **Recommendations** | Out of algorithmic scope, but architecturally: a separate offline/near-real-time pipeline consuming view/engagement events, producing a candidate list served from a low-latency store (Redis/feature store) at request time. Worth naming the **separation of the serving path from the model-training path** as the key architectural insight, even without discussing the ranking model itself. |

---

## 13. Architectural Alternatives — Pros & Cons

An expert architect's job is to show you *considered* alternatives, not just landed on one. Below are the major fork points in this design.

### 13.1 — Trigger mechanism: S3 Event Notifications vs. Kafka topic

| | S3 Event Notifications | Kafka topic (`video-uploaded`) |
|---|---|---|
| **Pros** | Zero extra infrastructure; built into S3; simple for a linear DAG (chunk → transcode → done) | Replay capability (re-process without re-upload); supports multiple independent consumer groups (e.g. chunker AND an analytics pipeline both react to the same event); natural backpressure via consumer lag monitoring; consistent with treating the pipeline as a durable, decoupled bus (as in the web crawler design) |
| **Cons** | No replay — if the chunker has a bug and you need to reprocess, you must re-trigger manually; single consumer pattern gets awkward with multiple downstream systems | Extra operational component to run/scale/monitor; overkill at this problem's actual scale (12-60 uploads/sec) |
| **When to choose which** | Small-to-medium scale, single linear pipeline, minimal reprocessing needs — **the right default for this problem's stated scale** | Needed once you have multiple independent consumers of the upload event, need replay for pipeline bugs, or scale is an order of magnitude higher |

### 13.2 — Transcoding compute: dedicated worker fleet vs. serverless (Lambda/Cloud Functions)

| | Dedicated worker fleet (EC2/containers, autoscaled) | Serverless (Lambda-style) |
|---|---|---|
| **Pros** | No per-invocation time limits (4K transcodes can be slow); predictable cost at sustained high volume; easier to use GPU-accelerated encoders | No idle cost, scales to zero between uploads; less operational overhead; good fit for the actual arrival rate here (~12-60/sec avg, bursty) |
| **Cons** | Pay for idle capacity during low-traffic hours unless carefully autoscaled | Execution time limits (historically ~15 min on Lambda) make long/4K transcodes awkward — may need to chunk the transcode job itself, or fall back to a worker fleet for large jobs; cold starts add latency |
| **When to choose which** | High, sustained upload volume (real YouTube scale) | Lower/spikier volume, or as the trigger/orchestration layer even if actual encoding runs on dedicated GPU workers |

### 13.3 — CDN strategy: Push vs. Pull

| | Push (proactively distribute to edge on upload) | Pull (cache-on-first-request, "pull-through cache") |
|---|---|---|
| **Pros** | No cold-miss latency for the first viewer in a region; good for content you know will be popular immediately (e.g. a major creator's new upload) | Only caches what's actually being watched — avoids wasting edge storage/bandwidth on unpopular content; simpler, this is what most CDNs do by default |
| **Cons** | Wastes edge bandwidth/storage pushing content nobody in that region ever watches — most videos have a long tail with near-zero views | First viewer per region always eats a cache-miss round trip to origin |
| **When to choose which** | **Pull is the correct default** for YouTube's actual content distribution (power-law view distribution — a small number of videos get almost all the views). Push only makes sense for a known-hot subset (trending page, a creator's day-of-release surge). |

### 13.4 — Metadata database: relational vs. NoSQL

Already covered in [§11](#11-component-5-metadata-store) — at this scale it's genuinely a toss-up. **Relational (Postgres)** wins if you expect to grow into comments/likes/subscriptions with real joins; **NoSQL (DynamoDB)** wins if you want to avoid ever thinking about schema migrations and are comfortable modeling access patterns (by-ID, by-user) up front via partition/sort keys and GSIs.

### 13.5 — Upload chunk stitching: client-driven vs. server-driven

| | Client uploads directly to S3 via pre-signed URLs (multipart upload) | Client uploads to app server, server relays/stitches to S3 |
|---|---|---|
| **Pros** | No video bytes ever pass through your app tier — dramatically lower compute/network cost on your servers; S3 handles stitching natively; this is the industry-standard pattern | — |
| **Cons** | Slightly more complex client SDK integration; requires trusting S3 event notifications (not the client) for completion state | Doubles network cost (client→server, server→S3); app tier becomes a bottleneck and a single point of failure for all uploads; **no real advantage** over the direct approach |
| **Verdict** | **Always prefer client-direct multipart upload** for large binary payloads — server-relay has essentially no upside here and is a common junior-level design smell. |

---

## 14. CAP Theorem Positioning

| Component | CAP Choice | Reasoning |
|---|---|---|
| Video upload / metadata propagation | **AP** | Explicitly called out in requirements — availability over consistency. A video being briefly invisible after upload is fine; the system being unavailable is not. |
| Raw video store (S3) | **AP** | Objects are immutable once written; no write conflicts. Durability + availability prioritized. |
| Transcoded chunk store (S3) | **AP** | Same reasoning; also append-only per (videoId, resolution, chunkIndex) key. |
| Video metadata DB | **AP** (tunable) | Use eventually-consistent reads for browsing/discovery; can use strongly-consistent reads for the uploader checking their own upload status if desired. |
| CDN cache | **AP** | Stale-for-a-short-TTL is completely acceptable; availability (fast delivery) is the entire point of the CDN's existence. |
| View-count aggregation | **AP** | Approximate, eventually-consistent counters are the norm at this scale (same principle as most large-scale counting systems). |
| Upload-completion coordination (if using a workflow engine / step function) | **CP** | The state machine tracking "has this video finished all N transcoding jobs" benefits from consistency — you don't want to double-trigger manifest finalization due to a split-brain read. |

---

## 15. Failure Modes & Mitigations

| Failure | Symptom | Detection | Mitigation |
|---|---|---|---|
| Client upload interrupted mid-multipart | Partial object in S3, metadata stuck `PENDING` | S3 multipart upload never receives "complete" call; TTL sweep | S3 auto-expires incomplete multipart uploads after a configured window (lifecycle policy); background sweep flags stale `PENDING` rows for cleanup/retry prompt to user |
| Transcoding job fails for one resolution | That resolution missing from manifest | Job failure event / DLQ | Don't block other resolutions; retry independently; publish partial manifest with available resolutions, backfill later |
| Chunker crashes mid-video | Some chunks missing, video stuck `CHUNKING` | Consumer lag / stuck-status sweep | Idempotent, resumable chunking keyed by `(videoId, chunkIndex)`; retry from last successful chunk, not from scratch |
| CDN cache miss storm (viral video, cold cache everywhere) | Origin (S3) suddenly hit with huge concurrent request volume | Origin request-rate spike alarm | Request coalescing at the CDN edge (many concurrent misses for the same object collapse into one origin fetch); pre-warm CDN for known high-anticipation uploads |
| Metadata DB hot partition (mega-viral video's row read constantly) | Elevated read latency on one partition/shard | Per-key request-rate metrics | Cache metadata for hot videos in Redis in front of the DB; CDN already absorbs most of the *content* read load, but metadata reads (title/description) can still spike |
| Pre-signed URL leaked/reused beyond intended scope | Unauthorized upload to a client's storage slot | Access logs, anomalous upload patterns | Short URL expiry (minutes, not hours); scope URLs to exact expected part size where the API supports it |
| Transcoder fleet under-provisioned during upload spike | Growing backlog of `TRANSCODING`-status videos | Queue depth / backlog age metric | Autoscale transcoder fleet on queue depth or CPU; prioritize lower resolutions first so *some* playable version ships fast even if 4K lags |
| Region-local S3/CDN outage | Viewers in one region can't stream | Regional error-rate monitoring | Multi-region S3 replication for popular/recent content; CDN fails over to a healthy origin region |

---

## 16. Scalability & Sharding

### What actually needs to scale (revisit capacity estimation)

- **Video Service (app tier)**: stateless → trivial horizontal scaling behind the API Gateway/load balancer
- **Chunker & Transcoder fleets**: stateless workers → autoscale on queue depth/CPU, exactly like the web crawler's fetcher/parser fleets
- **S3 / Blob storage**: effectively infinitely scalable, managed — not a design concern
- **Metadata DB**: at this problem's stated scale, a **single well-sized instance handles this for years**. If pushed to justify sharding: shard by `video_id`, add a GSI/secondary index on `user_id` for "all videos by creator"
- **CDN**: the component that matters most for *cost* at scale — this is where "add more capacity" has real budget implications, unlike S3/compute which scale cleanly

### The real scaling lesson of this problem

Unlike many system design problems where the database is the bottleneck, **YouTube's bottleneck is bandwidth and compute for transcoding, not storage or the metadata layer.** Correctly identifying *where the scaling pressure actually lives* — and explicitly saying "this part is not a bottleneck and I won't over-engineer it" — is a stronger signal than reflexively sharding everything.

---

## 17. Live Streaming (Extension Topic)

If the interviewer extends the problem to live streaming, name the core difference immediately: **you can no longer do asynchronous, offline transcoding** — there is no complete file to chunk after the fact. Key changes:

- **Ingest**: streamer's client encodes and pushes a continuous stream (RTMP is common) to an ingest server, not a completed file to S3
- **Real-time transcoding**: transcode each short segment (a few seconds) as it arrives, in near-real-time, into multiple bitrates — same multi-resolution idea, but pipelined instead of batch
- **Manifest updates continuously**: the manifest is a **live, growing** document (HLS calls this a "live playlist") — new segment URLs are appended as they become available, rather than being complete at generation time
- **Latency budget is much tighter** — end-to-end (broadcaster → viewer) latency target is seconds, not "500ms to first pixel of an already-uploaded video." This pushes toward smaller segments (1-2s) and protocols/extensions built for low-latency delivery (e.g. LL-HLS)
- **CDN caching is still critical** but TTLs must be very short since content is constantly appended

This is a strong "I understand the fundamentals deeply enough to reason about a variant, not just recite the VOD design" signal at Staff level.

---

## 18. Senior vs Staff Answer Differentiators

### Senior-level expectations

- Correctly identify the need for blob storage (S3) + separate metadata DB
- Recognize (often with interviewer hints) that large files can't go through a single POST request
- Understand at a high level that different resolutions are needed for different bandwidths, even without the word "transcoding"
- Get to basic chunking for playback, possibly with prompting
- Handle the obvious happy path end-to-end

### Staff-level expectations (additional)

| Topic | What Staff-level adds |
|---|---|
| Upload design | **Proactively** identifies the POST body-size limit and multipart upload solution — no interviewer hint required |
| Chunking | Explicitly distinguishes upload-chunking from playback-chunking as two different mechanisms with different sizing rationale |
| Transcoding | Proactively introduces the concept, describes it as a fan-out pipeline stage, discusses partial-failure handling per resolution |
| Trust boundaries | Recognizes that client-reported "upload complete" cannot be trusted; uses S3 notifications as source of truth |
| CDN | Connects CDN adoption directly back to the capacity estimation (egress bandwidth as dominant cost), not just "add a CDN because it's fast" |
| Streaming protocols | Can name HLS/DASH and describe *what problem they solve* (even without deep protocol knowledge) |
| Alternatives | Can discuss push vs. pull CDN strategy, Kafka vs. S3-event triggering, serverless vs. dedicated transcoding compute — with tradeoffs, not just a single answer |
| Failure modes | Proactively raises partial transcode failure, stuck-pipeline detection, idempotent retries, cache-miss storms |
| CAP theorem | States the AP-for-upload / AP-everywhere-except-coordination positioning unprompted, ties it back to the very first non-functional requirement |
| Extension handling | Can reason about live streaming as a variant, correctly identifying that offline/async transcoding is no longer possible |

### The single most important Staff differentiator

**Correctly identifying where the actual scaling pressure lives** (bandwidth/transcoding compute) **and explicitly not over-engineering the parts that don't need it** (metadata DB fits on one instance for years). A Staff engineer's judgment is measured as much by what they *don't* build as by what they do.

---

## 19. Interview Time Allocation

For a 45-minute system design session:

| Phase | Time | Focus |
|---|---|---|
| Requirements | 5 min | Functional (upload/watch) + non-functional (CAP, large files, low latency, scale) |
| Core entities & API | 5 min | Video vs. VideoMetadata split; naive API → identify size-limit flaw |
| High-level design | 5 min | Draw the happy path: upload → S3 → metadata; watch → metadata → S3 |
| Upload deep dive | 8 min | Multipart upload, pre-signed URLs, S3 notifications as source of truth |
| Chunk + transcode deep dive | 10 min | Two kinds of chunking, transcoder fan-out, partial-failure handling |
| Streaming/CDN deep dive | 8 min | Manifest files, ABR, CDN caching tied to bandwidth cost |
| Failure modes & alternatives | 4 min | Pick 2-3 from §13/§15, discuss tradeoffs concretely |

**What to cut if short on time:** metadata DB schema detail, adjacent features (comments/search/recs), live streaming extension. **Never cut:** the multipart upload redesign or the transcoding/chunking distinction — these are the load-bearing insights of the entire problem.

---

## 20. Quick-Reference Cheatsheet

```
KEY PATTERNS
────────────
Large file upload:   Pre-signed URLs + client-direct multipart upload to S3
                      (app tier never sits in the byte path)
Upload trust:         S3 event notification = source of truth, never trust
                      client-reported completion
Two chunkings:        Upload chunks (5-10MB, arbitrary offsets) vs.
                      Playback chunks (2-10s, key-frame aligned)
Transcoding:           Fan-out per resolution (240p...4K), partial-failure
                      tolerant, cheap resolutions first
Manifest file:         resolution → ordered chunk URL list; small, cacheable,
                      served from CDN
Adaptive bitrate:      Client re-measures bandwidth per chunk, switches
                      resolution mid-stream without interruption
CDN role:              Attacks the dominant cost (egress bandwidth), pull-
                      through cache is the default strategy

KEY NUMBERS
───────────
1M uploads/day    →  ~12/sec avg, ~40-60/sec peak
100M DAU          →  ~500M views/day, ~5,787/sec avg views
256 GB            →  max video size (real YouTube limit)
~1KB/video        →  metadata row size → NOT a scaling bottleneck
~25 PB/day        →  estimated egress bandwidth → THE dominant cost driver
500ms             →  target time-to-first-pixel

KEY SYSTEMS
───────────
Blob storage:      S3 / GCS — raw video + all transcoded chunks
Metadata DB:        Postgres or DynamoDB — trivial scale, pick based on
                    whether you expect relational growth (comments/likes)
Async trigger:      S3 event notifications (default) or Kafka (if you need
                    replay / multiple consumers)
Processing fleet:   Stateless chunker + transcoder workers, autoscaled on
                    queue depth
CDN:                 Pull-through cache at edge, holds hot chunks + manifests

CAP DECISIONS
─────────────
AP:  Video upload/metadata, S3 (raw + transcoded), CDN cache, view counts
CP:  Pipeline-completion coordination (if using a workflow/state machine)

STAFF-LEVEL SIGNALS TO HIT
───────────────────────────
✓ Proactively catch the POST body-size limit before being asked
✓ Correctly separate upload-chunking from playback-chunking
✓ S3 notifications, not client callbacks, as completion source of truth
✓ Partial transcode-failure handling (don't block on the slowest rendition)
✓ Tie CDN adoption explicitly back to the egress-bandwidth cost estimate
✓ Discuss push vs. pull CDN, Kafka vs. S3-event triggers, with tradeoffs
✓ Explicitly state where scaling pressure does NOT exist (metadata DB)
✓ Reason correctly about live streaming as a variant (no async transcode)
✓ Explicit CAP positioning tied back to the very first requirement stated
```

---

*Guide compiled for Senior & Staff-level system design interview preparation.*
*Topics: YouTube, Video Streaming, Large File Upload, Transcoding, CDN, CAP Theorem.*
*Source material: Hello Interview "Design YouTube" walkthrough transcript + expert architectural analysis and extensions.*
