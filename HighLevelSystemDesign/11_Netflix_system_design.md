# Netflix — System Design Interview Guide
### Senior & Staff Engineer Level

---

## Table of Contents

1. [Problem Statement & Scope](#1-problem-statement--scope)
2. [Requirements](#2-requirements)
3. [Capacity Estimation](#3-capacity-estimation)
4. [High-Level Architecture: Control Plane vs Data Plane](#4-high-level-architecture-control-plane-vs-data-plane)
5. [Component 1: Open Connect CDN (Data Plane)](#5-component-1-open-connect-cdn-data-plane)
6. [Component 2: Video Ingestion & Encoding Pipeline](#6-component-2-video-ingestion--encoding-pipeline)
7. [Component 3: Client & Adaptive Bitrate Playback](#7-component-3-client--adaptive-bitrate-playback)
8. [Component 4: Playback Control Plane (Entitlement, DRM, Manifest)](#8-component-4-playback-control-plane-entitlement-drm-manifest)
9. [Component 5: Metadata & Catalog Service](#9-component-5-metadata--catalog-service)
10. [Component 6: Recommendation & Personalization System](#10-component-6-recommendation--personalization-system)
11. [Component 7: User Profile & Viewing History](#11-component-7-user-profile--viewing-history)
12. [Component 8: Search Service](#12-component-8-search-service)
13. [Component 9: Microservices Platform (Gateway, Discovery, Caching, Chaos Engineering)](#13-component-9-microservices-platform-gateway-discovery-caching-chaos-engineering)
14. [CAP / PACELC Theorem Positioning](#14-cap--pacelc-theorem-positioning)
15. [Failure Modes & Mitigations](#15-failure-modes--mitigations)
16. [Scalability & Sharding](#16-scalability--sharding)
17. [Freshness & Consistency Considerations](#17-freshness--consistency-considerations)
18. [Senior vs Staff Answer Differentiators](#18-senior-vs-staff-answer-differentiators)
19. [Interview Time Allocation](#19-interview-time-allocation)
20. [Quick-Reference Cheatsheet](#20-quick-reference-cheatsheet)

---

## 1. Problem Statement & Scope

**Netflix** is a global video-on-demand (VOD) streaming platform. The interview version of "design Netflix" is really two coupled problems wearing one trenchcoat:

1. **Deliver bytes** — get an encoded video stream to hundreds of millions of heterogeneous devices with low startup latency and minimal rebuffering, at a bandwidth cost that doesn't bankrupt the business.
2. **Decide what to show** — personalize catalog discovery (homepage rows, ranking, search, even artwork) so that a catalog of thousands of titles feels tailored to each of hundreds of millions of individual tastes.

Scope this out loud early: are you designing playback, personalization, the encoding pipeline, or the whole platform? A 45-minute session cannot do all four with real depth — pick a primary focus (playback/CDN is the most common ask) and treat the rest as breadth.

### What the interviewer is really testing

- Can you cleanly separate **control plane (decisions) from data plane (bytes)** — and explain why that separation is the reason a backend outage doesn't necessarily stop a stream that's already playing?
- Do you understand **CDN architecture** well enough to go beyond "put a CDN in front of it" — proactive placement, not just reactive caching?
- Do you understand the **encoding tradeoffs** (bitrate ladders, codecs, perceptual quality) that make streaming at this scale economically viable?
- Can you design **personalization as a systems problem** (offline/online split, experimentation infrastructure) rather than hand-waving "ML recommends stuff"?
- Do you design explicitly for **partial failure** in a system built from hundreds of interdependent microservices?

---

## 2. Requirements

### Functional Requirements

| Requirement | Notes |
|---|---|
| Browse a personalized homepage | Rows, ranking within rows, personalized artwork |
| Search the catalog | Typeahead, personalized ranking of results |
| Play a title with adaptive quality | Minimal startup delay, minimal rebuffering |
| Resume playback across devices | Pause on TV, resume on phone |
| Multiple profiles per account | Independent history, recommendations, parental controls |
| Support heterogeneous devices | Smart TVs, consoles, mobile, browsers — different codec/DRM/resolution support |
| Offline download | Encrypted local storage, expiring licenses |
| Multi-language audio/subtitles | Per-title, per-region availability |
| Regional content licensing | A title may be unavailable or windowed differently by country |

### Non-Functional Requirements

| Requirement | Target |
|---|---|
| Scale | ~300M+ subscriber accounts; tens of millions of concurrent streams at peak |
| Startup latency | Time-to-first-frame in the low seconds |
| Availability | 99.99%+; active-active across regions |
| Global reach | Content and rights vary by country; low-latency delivery on every continent |
| Cost efficiency | Egress bandwidth is the dominant operating cost — the CDN strategy is a cost-engineering problem as much as a performance one |
| Encoding efficiency | Maximize perceptual quality per bit delivered |
| Graceful degradation | A single backend microservice failing should degrade a feature, not break playback or the whole homepage |

---

## 3. Capacity Estimation

*(Treat every figure below as an illustrative reasoning tool, not a pinned fact — stay comfortably vague under follow-up pressure on exact numbers.)*

```
Assume: ~300M subscriber accounts globally

Concurrent streams at peak (illustrative ~5% concurrency):
  300,000,000 × 0.05 ≈ 15,000,000 concurrent streams

Blended average bitrate (mix of SD/HD/4K/mobile):
  ~3 Mbps average across the device/quality mix

Peak aggregate egress:
  15,000,000 streams × 3 Mbps ≈ 45 Tbps

Control-plane fan-out (homepage load):
  ~100M daily active users × ~2 homepage loads/day
  × ~20 backend calls per load (metadata, rec rows, artwork, profile, etc.)
  → tens of millions of backend RPS in aggregate, globally distributed

Catalog storage (illustrative):
  ~15,000 hours of unique title runtime
  × multiple resolutions × multiple codecs (H.264/VP9/AV1)
  × multiple bitrate rungs × audio/subtitle tracks
  → each hour of source becomes 1,000+ hours of encoded renditions
```

**Key insight:** unlike the web-crawler problem (where *storage* is the dominant cost), here **egress bandwidth is the dominant cost**. 45 Tbps of peak traffic served from generic third-party CDN capacity, at generic transit pricing, does not scale economically — this single constraint is *why* Netflix built and operates its own CDN (Open Connect) instead of only buying CDN capacity. Capacity estimation here should conclude with an architectural consequence, not just a number.

---

## 4. High-Level Architecture: Control Plane vs Data Plane

The single most important framing decision for this design: **separate the low-QPS, decision-making control plane from the massive-QPS, byte-delivery data plane.** They have different scaling laws, different consistency needs, and — critically — different failure blast radii.

```
                         ┌────────────────────────────┐
                         │         Client Device       │
                         │  (TV / Mobile / Browser)    │
                         └───────────┬─────────┬───────┘
                    CONTROL PLANE    │         │   DATA PLANE
                (decisions, low QPS) │         │ (bytes, massive QPS)
                                     │         │
                  ┌──────────────────▼──┐   ┌──▼────────────────────┐
                  │  API Gateway (Zuul)  │   │  Open Connect          │
                  │                      │   │  Appliance (in ISP)    │
                  └──────────┬───────────┘   └──────────┬─────────────┘
                             │                            │ cache miss
        ┌────────────────────┼────────────────────┐       │
        │                    │                     │       │
┌───────▼──────┐   ┌─────────▼────────┐   ┌────────▼──────┐│
│ Playback API │   │ Recommendation   │   │ Metadata /    ││
│ (entitlement,│   │ Service          │   │ Catalog       ││
│  DRM, manifest)  │ (row/rank/art)   │   │ Service       ││
└───────┬──────┘   └─────────┬────────┘   └────────┬──────┘│
        │                    │                     │       │
        └────────────────────┼─────────────────────┘       │
                EVCache / Cassandra / Elasticsearch         │
                                                             │
                                              ┌──────────────▼─────────────┐
                                              │  Open Connect Core Site /  │
                                              │  S3 Origin (encoded video) │
                                              └────────────────────────────┘
```

**Why this split matters (Staff-level framing):** once a client has a manifest and a license, every subsequent video chunk is fetched **directly from the CDN** — it never touches the control-plane microservices again for that stream. This means a control-plane incident (say, the recommendation service falling over) can degrade *new* homepage loads while having **zero effect on the millions of streams already playing**. Interviewers are listening for whether you understand this decoupling, because it's the reason Netflix's architecture tolerates deep, cascading backend failures without a global outage.

---

## 5. Component 1: Open Connect CDN (Data Plane)

This is the most distinctively "Netflix" piece of the system, and it deserves the most airtime.

### Why not just use a third-party CDN?

At ~45 Tbps of peak egress (see capacity estimation), buying that capacity at generic CDN transit pricing is not economically viable at scale, and generic CDNs are typically **reactive**: they cache whatever gets requested, evicting on LRU, with no foreknowledge of what's about to go viral. Netflix instead built **Open Connect**: a purpose-built CDN of appliances (OCAs — Open Connect Appliances) that Netflix ships for free to ISPs, who rack them inside their own networks (or at internet exchange points).

### Proactive Fill — the core architectural idea

```
                     ┌─────────────────────────┐
                     │  Demand Prediction Model │
                     │  (per-title, per-region) │
                     └────────────┬─────────────┘
                                  │ nightly fill plan
                                  ▼
   ┌────────────────────────────────────────────────────┐
   │               Open Connect Core Sites                │
   │        (regional hubs — hold the full catalog)        │
   └───────────────────────┬────────────────────────────┘
                            │  off-peak proactive push
       ┌────────────────────┼────────────────────┐
       ▼                    ▼                     ▼
 ┌───────────┐        ┌───────────┐         ┌───────────┐
 │ OCA @ ISP │        │ OCA @ ISP │         │ OCA @ IXP │
 │  (edge)   │        │  (edge)   │         │  (edge)   │
 └─────┬─────┘        └─────┬─────┘         └─────┬─────┘
       │                    │                     │
     end users           end users             end users
```

Unlike a typical CDN that fills its cache **reactively** (on first request, then evicts LRU), Open Connect fills edge appliances **proactively**, overnight, during regional off-peak hours, based on a predicted-popularity model per title per region. By the time users wake up, the content they're statistically likely to watch that day is already sitting on the appliance closest to them.

**Consequence:** in steady state, the overwhelming majority of streams are served entirely from an OCA inside the requesting user's own ISP — minimal transit hops, minimal transit cost, maximum throughput and lowest latency to the end user.

### Steering: routing is a control-plane decision, not a CDN decision

The OCAs themselves are intentionally "dumb" — they don't run request-routing logic. **Steering** (which OCA a given client should hit) is decided by a control-plane service in AWS, using near-real-time signals: OCA health, current load, network path quality, and predicted best route. This is another instance of the control/data plane split: the *decision* of where to fetch bytes from is control plane; the *fetching* is data plane.

### Fallback tiers

```
Tier 1: Edge OCA at ISP/IXP        → fastest, cheapest, default path
Tier 2: Open Connect Core Site     → regional hub, used on edge cache miss
Tier 3: S3 Origin                  → source of truth, rarely hit directly
```

A cache miss at the edge is rare in steady state (thanks to proactive fill) but must degrade gracefully — falling back a tier at a time rather than failing the request.

### Senior vs Staff on this component

A Senior answer says "put a CDN in front of the video." A Staff answer explains **why a reactive, pull-based CDN doesn't fit this cost/scale profile**, and describes a **proactive, prediction-driven fill strategy with a separate steering decision** made in the control plane. That distinction — reactive caching vs. proactive placement — is the single most important thing to get right in this whole guide.

---

## 6. Component 2: Video Ingestion & Encoding Pipeline

### Pipeline overview

```
Studio Master File (mezzanine, near-lossless)
        │
        ▼
┌────────────────────────────────────────┐
│  Chunking                                │  split into N parallel segments
└──────────────┬───────────────────────────┘
               │
┌──────────────▼───────────────────────────┐
│  Per-Title / Per-Shot Complexity Analysis │  → optimal bitrate–resolution
│                                            │    ladder, driven by VMAF
└──────────────┬───────────────────────────┘
               │
┌──────────────▼───────────────────────────┐
│  Parallel Encode Workers                  │  → H.264 / VP9 / AV1 × N renditions
└──────────────┬───────────────────────────┘
               │
┌──────────────▼───────────────────────────┐
│  QC Gate (automated + human review)       │
└──────────────┬───────────────────────────┘
               │
┌──────────────▼───────────────────────────┐
│  Packaging + DRM encryption + Manifest    │
└──────────────┬───────────────────────────┘
               │
         ┌─────▼─────┐
         │ S3 Origin │──▶ proactive push to Open Connect (see Component 1)
         └───────────┘
```

### Per-title / per-shot encoding (Staff-level detail)

A naive design applies one fixed bitrate ladder (e.g., 235/375/750/1750/3000/5800 kbps at fixed resolutions) to every title. Netflix instead analyzes each title's — and in more advanced pipelines, each *shot's* — visual complexity and computes a custom convex-hull ladder: the set of resolution/bitrate pairs that maximizes perceptual quality (measured by **VMAF**, Netflix's own perceptual video quality metric) for that specific content. A static talking-head scene needs far less bitrate than a high-motion action sequence to look equally good; a fixed ladder wastes bits on the former and under-serves the latter.

### Chunk-based parallel encoding

Splitting the source into small chunks and encoding each on a separate worker gives massive parallelism (turnaround measured in minutes, not hours, for a full catalog title), and — importantly for the failure-modes discussion — makes a single chunk's encode failure a cheap, isolated retry instead of restarting an entire title's encode job.

### Multiple codecs

H.264 for broad device compatibility; VP9/AV1 for capable devices at meaningfully better compression (lower bandwidth for the same perceptual quality). This multiplies the rendition combinatorics (codec × resolution × bitrate × audio track × subtitle track), which is why the "storage per hour of source" multiplier in the capacity section is so large — and why old/cold titles are natural candidates for tiered/cheaper storage, echoing the same hot/cold tiering principle used in the [Web Crawler guide's storage layer](#11-component-7-storage-layer) and the general [Sharding Strategies guide](#).

### QC gate

Automated checks (audio sync, black-frame detection, VMAF threshold) plus human spot-review before a rendition is allowed to publish. A bad encode reaching Open Connect's proactive fill would propagate to edge caches globally before anyone noticed — the QC gate exists specifically to prevent that blast radius.

---

## 7. Component 3: Client & Adaptive Bitrate Playback

### Manifest-driven playback

The client requests a manifest (DASH or HLS style) listing every available rendition for the title. Video is chunked into short segments (2–10s), each independently encoded at every bitrate/resolution rung, keyframe-aligned across renditions so the client can switch bitrate at any segment boundary without a visible glitch.

### Adaptive bitrate (ABR) algorithm

The client continuously estimates two signals and picks the next segment's bitrate accordingly:

```
Signal 1: Throughput estimate    → recent segment download speed
Signal 2: Buffer occupancy       → seconds of video currently buffered

Hybrid ABR logic (conceptual):
  if buffer is low AND throughput is uncertain:
       choose a conservative (lower) bitrate — protect against rebuffering
  if buffer is healthy AND throughput is strong:
       ramp up toward the highest quality the connection supports
```

Startup strategy: begin at a low, safe rendition to minimize time-to-first-frame, then ramp up once buffer and throughput are established. This tradeoff — fast start vs. immediate top quality — is a deliberate UX decision, not an oversight.

### DRM handshake

Before decrypting the first segment, the client performs a license request to a separate licensing service (Component 8's neighbor — see Playback Control Plane), using the device's native DRM system (Widevine, PlayReady, or FairPlay depending on platform). This handshake happens once per session, not once per chunk — another reason a brief control-plane blip doesn't kill an already-playing stream.

---

## 8. Component 4: Playback Control Plane (Entitlement, DRM, Manifest)

When the user presses play, the client hits the **Playback API** — the highest-stakes control-plane service in the whole system, because everything downstream (which CDN, which renditions, whether playback is even allowed) is decided here.

```
Client "play" request
        │
        ▼
┌─────────────────────────────────────────┐
│  1. Entitlement check                     │  is this title licensed for this
│                                            │  account's region right now?
├─────────────────────────────────────────┤
│  2. Device capability negotiation         │  which codecs/DRM/resolutions
│                                            │  does this device support?
├─────────────────────────────────────────┤
│  3. CDN selection (steering)              │  which Open Connect appliance?
├─────────────────────────────────────────┤
│  4. DRM license handshake                 │  issue a short-lived license
├─────────────────────────────────────────┤
│  5. Signed manifest generation            │  URLs pointing at the selected OCA
└─────────────────────────────────────────┘
        │
        ▼
Client begins fetching segments directly from the CDN (data plane)
```

### Why entitlement gets special treatment

Regional content licensing is a **legal and contractual constraint**, not just a UX preference. Serving a title outside its licensed region is a real business/legal exposure. This is why the entitlement check is one of the few places in this otherwise heavily availability-biased system where **correctness is prioritized over availability** — see the CAP/PACELC table below.

Backed by a low-latency store (EVCache in front of Cassandra) since this check sits directly on the critical path of every play request.

---

## 9. Component 5: Metadata & Catalog Service

Holds the descriptive layer: title metadata (cast, genre, season/episode structure), artwork variants, and per-region availability windows driven by licensing contracts.

- **Read-heavy, write-light**: metadata changes far less often than it's read, making this a textbook heavy-caching candidate — cached aggressively behind EVCache, with cache responses themselves sometimes served from CDN edge for anonymous/generic parts of the catalog.
- **Partitioned by title ID** in the backing store (Cassandra or a document store), enabling efficient per-title and per-region queries.
- **Region-gated availability** ties directly into the entitlement check in the Playback Control Plane — metadata may *show* a title exists globally while entitlement independently governs whether *this* account in *this* region can actually press play.

---

## 10. Component 6: Recommendation & Personalization System

A Senior answer treats this as "an ML model recommends titles." A Staff answer decomposes it into a small ensemble of distinct systems:

```
Kafka: viewing-events (play, pause, seek, complete, rating)
        │
        ▼
┌──────────────────────────────┐        ┌───────────────────────────────┐
│  Offline / Batch (Spark)      │        │  Online / Real-time            │
│  - collaborative filtering    │        │  - session context              │
│  - embeddings, model training │        │  - "just watched" signal        │
└───────────────┬────────────────┘        └────────────────┬────────────────┘
                │  nightly                                   │  live
                ▼                                            ▼
        ┌─────────────────────────────────────────────────────────┐
        │              Precomputed + Blended Recommendations         │
        │  Row selection  │  Ranking within row  │  Artwork choice   │
        └───────────────────────────┬─────────────────────────────┘
                                    │ served on homepage load
                                    ▼
                            EVCache (low-latency reads)
```

### Three distinct personalization surfaces (Staff-level detail)

1. **Row selection** — which categories appear at all ("Continue Watching," "Trending Now," "Because You Watched X") and in what order.
2. **Ranking within a row** — which titles appear first inside a given row.
3. **Artwork personalization** — even the *thumbnail image* shown for the same title can differ per profile, chosen from a set of candidate images based on which image historically drives the highest engagement for that user's taste cluster. This is a genuinely non-obvious detail that signals real familiarity with the system rather than a generic "Netflix uses ML" answer.

### Offline/online split

Heavy model training (collaborative filtering, embeddings) runs as scheduled batch jobs (Spark) over the viewing-events history warehoused from Kafka — this can tolerate hours of staleness. Serving must be low-latency (sub-100ms) at homepage-load time, so results are precomputed and cached, then blended with a thin layer of real-time session signal (what you just watched five minutes ago) so "Continue Watching" doesn't feel stale even though the bulk of the recommendation model is a day old.

### Experimentation is itself an architectural component

Every ranking or row-selection strategy change ships behind an **A/B testing / experimentation platform** with holdout groups, not as a direct swap. At Staff level, call this out explicitly: personalization systems at this scale are validated continuously in production, not just offline — this is as much a piece of the architecture as the model-serving path itself.

*(This event pipeline mirrors the general durable-bus pattern described in the Kafka guide, and the low-latency serving layer parallels the Distributed Key-Value Store guide's read-path reasoning.)*

---

## 11. Component 7: User Profile & Viewing History

- **Multiple profiles per account**, each with independent history, recommendations, and (optionally) parental controls.
- **Resume playback position** is written frequently during playback (every few seconds) but read rarely (once, when the title is reopened) — a write-heavy, latency-sensitive, low-durability-requirement workload. Losing the last few seconds of resume position is a minor UX blip, not a correctness problem, which makes this a strong candidate for an **AP** store with eventual, last-write-wins convergence rather than anything requiring coordination.
- **Cross-device sync**: pause on the TV, resume on the phone — this requires the position update to propagate globally within a short SLA window, but does not require strict consistency; a stale-by-a-few-seconds resume point is an acceptable tradeoff for availability and low write latency.

---

## 12. Component 8: Search Service

Typeahead and full search over title, cast, and genre metadata, with a personalization layer blended into ranking similar in spirit to Component 6. Backed by an inverted index (Elasticsearch or equivalent), sharded for scale. Lower QPS than the homepage/playback path, but still needs sub-100ms responses to feel responsive while typing.

---

## 13. Component 9: Microservices Platform (Gateway, Discovery, Caching, Chaos Engineering)

Netflix's control plane is not one service — it's hundreds of microservices, and the platform-level concerns of holding that together are themselves interview-worthy.

### Edge & discovery

- **API Gateway (Zuul-style)**: single entry point from clients, handles routing, auth, and edge-level rate limiting.
- **Service discovery**: a registry that lets services find each other's healthy instances dynamically as the fleet scales up/down. Netflix's own service-discovery layer (Eureka) is a well-known public example of an **explicit AP design choice**: it favors availability over consistency — a stale registry entry (routing to an instance that's since died, caught by client-side retry) is considered a far smaller problem than a discovery outage blocking all inter-service calls. This is a good concrete example to cite when discussing CAP tradeoffs.

### Fan-out and graceful degradation

A single homepage load fans out to dozens of backend microservices (metadata, recommendations, artwork, profile, entitlement, etc.). At this fan-out depth, **circuit breakers at every dependency edge are not optional** — if the recommendation service is slow or down, the homepage should render with a cached or generic set of rows rather than fail the entire page load. This is the same pattern covered in depth in the [Circuit Breaker Pattern guide](#) — Netflix is in fact the origin case study for that pattern (Hystrix) in most interview prep material, so drawing the explicit connection is a strong signal.

### Caching

**EVCache** — Netflix's globally distributed caching layer built on Memcached — sits in front of most read-heavy, latency-sensitive backing stores (metadata, profile, entitlement) to keep the fan-out path fast and to shield Cassandra from read load it was never sized for.

### Chaos engineering as an architectural philosophy

Rather than only designing for failure reactively, Netflix continuously and proactively verifies failure tolerance **in production**:

```
Chaos Monkey   → randomly terminates individual instances
Latency Monkey → injects artificial latency into service calls
Chaos Kong     → simulates an entire AWS region failure
```

This is a genuine Staff-level talking point: it reframes resilience from "we handled the failure modes we thought of" to "we continuously verify resilience against failure modes in the live system," backed by active-active multi-region deployment so that a Chaos-Kong-style regional failure has somewhere to fail over to.

---

## 14. CAP / PACELC Theorem Positioning

As with any distributed system at this scale, CAP/PACELC must be reasoned about **per component**, not for "Netflix" as a whole — the system deliberately mixes AP-biased and CP-biased components. *(See the dedicated CAP/PACELC Theorem guide for the full framework this table applies.)*

| Component | CAP Choice | PACELC (no-partition branch) | Reasoning |
|---|---|---|---|
| Open Connect CDN (video bytes) | **AP** | Latency favored | Content bytes are immutable once published; availability and low latency dominate — there's no "staleness" risk in serving cached video bytes |
| Entitlement / licensing check | **CP** | Consistency favored | Serving a title outside its licensed region is a legal/contractual risk; briefly denying playback is safer than an inconsistent "allow" |
| Metadata / catalog service | **AP** | Latency favored | Stale artwork or a slightly outdated synopsis is a minor UX issue, not a correctness problem — cache aggressively |
| Recommendation system | **AP** | Latency favored | Inherently eventually consistent by design (batch-trained); a day of model staleness is an accepted tradeoff |
| Viewing history / resume position | **AP** | Latency favored | Write-heavy, rarely read; last-write-wins convergence is acceptable |
| Profile / account / billing data | **CP** | Consistency favored | Billing state and access control must not be inconsistent across replicas |
| Service discovery (Eureka-style) | **AP** | Latency favored | A stale registry entry (retried client-side) is far cheaper than a discovery outage blocking all inter-service calls |
| Coordination/config (ZooKeeper/etcd-style, where used) | **CP** | Consistency favored | Shard assignment / leader election must be globally agreed upon |

---

## 15. Failure Modes & Mitigations

| Failure | Symptom | Detection | Mitigation |
|---|---|---|---|
| Viral/unexpected demand spike | Edge OCA cache miss storm, origin overload | Cache-miss rate spike on an OCA/region | Tiered fallback (edge → core site → origin); surge capacity headroom at core sites; pre-fill for *known* release dates reduces this for scheduled drops |
| Backend microservice failure (e.g., recommendation service down) | Homepage rows fail to load | Circuit breaker trip rate, error rate per dependency | Circuit breaker + fallback to cached/generic rows; never fail the whole page for one dependency |
| Cache stampede after deploy or cold EVCache | Backing store (Cassandra) suddenly overloaded | Backing-store latency/error spike right after a deploy | Request coalescing, staggered TTLs with jitter, warm-up before traffic cutover — this is the same failure signature as a thundering-herd cache stampede in any read-heavy service |
| DRM license service outage | New playback starts blocked | License request error rate | Regional license service replicas; already-playing sessions are unaffected since license was issued once at session start, not per-chunk |
| Encoding pipeline job failure | Missing rendition for a title | QC gate failure, missing chunk in pipeline output | Chunk-level retry (cheap, isolated); QC gate blocks a bad encode from ever reaching proactive CDN fill |
| Region-wide AWS outage | All control-plane services in one region unreachable | Region health checks, elevated error rate region-wide | Active-active multi-region deployment; traffic steering shifts control-plane load to healthy regions; regularly exercised via Chaos Kong |
| OCA hardware failure at an ISP site | Elevated latency/errors for users near that appliance | OCA health check failure | Steering routes around the unhealthy appliance to the next-best OCA or core site |
| Clock/consistency skew on resume position | User resumes from a slightly stale position on a second device | N/A — accepted tradeoff | Last-write-wins with timestamp; treated as acceptable UX cost, not a bug to eliminate with coordination |
| Catalog licensing window not honored precisely | Title remains playable after contract expiry | Compliance/legal audit, entitlement mismatch alert | Event-driven deactivation trigger tied to entitlement service, not reliance on cache TTL expiry alone (see Freshness section) |

---

## 16. Scalability & Sharding

| Component | Scaling Strategy |
|---|---|
| Open Connect | Scale by **geographic placement** — add appliances at more ISP/IXP locations, not just more compute; this is a distinct scaling axis from typical horizontal compute scaling |
| Encoding pipeline | Scale out parallel encode workers per chunk; embarrassingly parallel |
| Cassandra (profile, entitlement, metadata) | Partition by natural key (`userID` for profile/history, `titleID` for metadata); add nodes to the consistent-hash ring — see the general [Sharding Strategies guide](#) for partitioning tradeoffs |
| EVCache | Sharded and replicated across availability zones and regions |
| Kafka (viewing-events pipeline) | Partitioned by `userID` or event type; scale consumer groups independently for feature computation — see the [Kafka guide](#) |
| Recommendation compute (Spark) | Scales independently of the serving path — batch training cluster size is decoupled from read-path QPS |
| API Gateway | Stateless; scale horizontally behind a load balancer |

---

## 17. Freshness & Consistency Considerations

Different pieces of a single homepage load have **genuinely different freshness requirements**, and conflating them is a common Senior-level gap:

| Data | Acceptable staleness | Mechanism |
|---|---|---|
| Recommendation rows (general) | Up to ~1 day | Nightly batch retrain |
| "Continue Watching" row | Near-real-time (minutes) | Blended online signal on top of the batch model |
| Metadata/artwork updates | Minutes to hours | Push-based cache invalidation over TTL alone, so a new season or refreshed artwork doesn't wait out a long TTL |
| Catalog licensing expiry | **Must be exact**, not best-effort | Event-driven deactivation trigger tied to the entitlement service — a title staying live past contract end is a legal exposure, so this cannot rely on cache TTL drift |
| Resume position | Seconds to low minutes | Eventual, last-write-wins propagation |

The general principle: **treat freshness as a per-data-type design decision, not a single global cache-TTL policy.** A system that applies one TTL uniformly either over-invalidates cheap-to-cache data or under-invalidates legally sensitive data — and an interviewer probing this area is specifically listening for whether you'll spot that licensing expiry is different in kind, not just degree, from artwork staleness.

---

## 18. Senior vs Staff Answer Differentiators

### Senior-level expectations

- Correctly identifies the major components: CDN, transcoding/encoding, catalog/metadata service, recommendation service, playback API
- Knows adaptive bitrate streaming exists (mentions DASH/HLS)
- Basic capacity math (concurrent streams, bandwidth)
- Basic failure handling (retries, caching, "add more servers")
- Says "Netflix uses ML for recommendations"

### Staff-level expectations (additional)

| Topic | What Staff-level adds |
|---|---|
| Overall framing | Explicit control-plane / data-plane separation, and *why* it bounds the blast radius of backend failures |
| CDN | Proactive, prediction-driven Open Connect fill vs. reactive third-party CDN caching — with steering as a separate control-plane decision from the "dumb" edge appliance |
| Encoding | Per-title/per-shot bitrate ladder optimization (VMAF-driven) instead of a fixed ladder; chunk-parallel encoding; QC gate as a blast-radius control |
| Recommendation | Decomposed into row selection, in-row ranking, and artwork personalization as three distinct surfaces; explicit offline/online split; experimentation platform as an architectural component |
| CAP/PACELC | Explicit per-component reasoning, including a real, named example (Eureka's AP-by-design choice) rather than generic "some parts are AP, some are CP" |
| Resilience | Chaos engineering as continuous, in-production verification (Chaos Monkey/Kong), not just theoretical failure-mode enumeration |
| Fan-out | Circuit breakers at every dependency edge with graceful degradation (cached/generic fallback) rather than failing the whole page for one slow dependency |
| Entitlement | Recognized as a legally-bounded CP component, explicitly distinct from the mostly-AP rest of the system |
| Freshness | Recognizes that freshness requirements differ *in kind* across data types on the same page (licensing expiry vs. artwork vs. recommendations) |

### The single most important Staff differentiator

**Explaining why Open Connect's proactive fill model exists, and tying it back to the capacity/cost math from the estimation phase.** Almost every candidate can say "CDN." Very few connect the dots from "45 Tbps of peak egress makes generic reactive CDN caching uneconomical" to "therefore Netflix predicts demand and pre-positions content at the edge before it's requested, with routing decided separately in the control plane." That chain of reasoning — cost constraint → architectural consequence — is exactly the kind of proactive, first-principles thinking that separates a Staff answer from a Senior one.

---

## 19. Interview Time Allocation

For a 45-minute system design session:

| Phase | Time | Focus |
|---|---|---|
| Requirements & scope | 5 min | Confirm functional + non-functional; pick a primary focus (playback vs. personalization) |
| Capacity estimation | 5 min | Concurrent streams, egress bandwidth, fan-out RPS — conclude with the cost-driven architectural consequence |
| High-level architecture | 5 min | Draw the control-plane / data-plane split explicitly before naming individual services |
| Open Connect CDN (deep dive) | 8 min | Proactive fill, steering, fallback tiers — this is the highest-signal component |
| Encoding pipeline | 6 min | Per-title/shot ladder, chunked parallel encode, QC gate |
| Playback control plane + entitlement/DRM | 5 min | Entitlement as a CP component; manifest/license handshake |
| Recommendation system | 7 min | Offline/online split; row/rank/artwork decomposition; experimentation platform |
| Failure modes & resilience | 5 min | Chaos engineering philosophy, circuit breakers at fan-out edges, region failover |
| CAP table & wrap-up (if time) | 4 min | Fold per-component CAP reasoning in as you go if time is short |

**What to cut if short on time:** search service depth, exact storage schema detail. **Never cut:** the control-plane/data-plane framing and the Open Connect deep dive — these carry the most Staff-level signal per minute spent.

---

## 20. Quick-Reference Cheatsheet

```
CORE FRAMING
────────────
Control plane:  decisions, low QPS, mostly AP except entitlement (CP)
Data plane:     bytes, massive QPS, served by Open Connect CDN
Key insight:    control-plane outage ≠ playback outage — already-playing
                streams pull bytes directly from the CDN, not the backend

KEY ALGORITHMS / TECHNIQUES
────────────────────────────
CDN fill:        Proactive, prediction-driven push (not reactive pull-cache)
Encoding:        Per-title/per-shot bitrate ladder, VMAF-optimized convex hull
ABR:             Hybrid buffer-occupancy + throughput-estimate bitrate selection
Recommendation:  Offline batch (Spark) + online real-time blend
                 Row selection │ In-row ranking │ Artwork personalization
Resilience:      Chaos Monkey (instance) │ Latency Monkey (latency) │ Chaos Kong (region)

KEY NUMBERS (illustrative — stay vague under pressure)
─────────────────────────────────────────────────────
~300M subscriber accounts
~15M concurrent streams at peak (illustrative 5% concurrency)
~3 Mbps blended average bitrate
~45 Tbps peak aggregate egress  →  the reason Open Connect exists
Tens of millions of backend RPS at peak (homepage fan-out)

KEY SYSTEMS
───────────
Data plane:      Open Connect (OCAs at ISPs/IXPs) + Core Sites + S3 origin
Control plane:   API Gateway (Zuul-style) + microservices fleet
Discovery:       Eureka-style registry — explicitly AP by design
Caching:         EVCache (Memcached-based, globally distributed)
Event pipeline:  Kafka (viewing-events → batch training + online features)
Resilience:      Circuit breakers at every fan-out edge (Hystrix-style)

CAP DECISIONS
─────────────
AP:  Open Connect CDN, metadata/catalog, recommendations, viewing history, service discovery
CP:  Entitlement/licensing check, profile/billing/account data, coordination/config

STAFF-LEVEL SIGNALS TO HIT
───────────────────────────
✓ Control-plane / data-plane separation stated explicitly, early
✓ Proactive Open Connect fill vs. reactive CDN caching, tied back to cost math
✓ Steering as a control-plane decision separate from "dumb" edge appliances
✓ Per-title/per-shot VMAF-driven encoding ladder, not a fixed ladder
✓ Recommendation decomposed into row / rank / artwork, not one monolith
✓ Named, real CAP example (Eureka's AP-by-design choice)
✓ Entitlement singled out as the CP exception in an otherwise AP-heavy system
✓ Chaos engineering framed as continuous production verification, not theory
✓ Circuit breakers + graceful degradation at every fan-out edge
✓ Freshness requirements differentiated by data type, not one global TTL
```

---

*Guide compiled for Senior & Staff-level system design interview preparation.*
*Topics: Netflix, Video Streaming, CDN Architecture, Adaptive Bitrate, Personalization Systems, Microservices Resilience.*
*Cross-references: Circuit Breaker Pattern guide, Kafka guide, CAP/PACELC Theorem guide, Sharding Strategies guide, Distributed Key-Value Store guide.*
