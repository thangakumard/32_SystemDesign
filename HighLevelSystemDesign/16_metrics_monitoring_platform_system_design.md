# Metrics Monitoring Platform (Datadog-style) — System Design Interview Guide
### Senior & Staff Engineer Level

---

## Table of Contents

1. [Problem Statement & Scope](#1-problem-statement--scope)
2. [Requirements](#2-requirements)
3. [Capacity Estimation](#3-capacity-estimation)
4. [High-Level Architecture](#4-high-level-architecture)
5. [Component 1: Agent / Collector](#5-component-1-agent--collector)
6. [Component 2: Ingestion Gateway](#6-component-2-ingestion-gateway)
7. [Component 3: Message Bus (Kafka)](#7-component-3-message-bus-kafka)
8. [Component 4: Stream Aggregation & Rollup Pipeline](#8-component-4-stream-aggregation--rollup-pipeline)
9. [Component 5: Time-Series Storage Engine](#9-component-5-time-series-storage-engine)
10. [Component 6: Metadata / Tag Index Service](#10-component-6-metadata--tag-index-service)
11. [Component 7: Query Engine](#11-component-7-query-engine)
12. [Component 8: Alerting Engine](#12-component-8-alerting-engine)
13. [Component 9: Dashboarding & API Layer](#13-component-9-dashboarding--api-layer)
14. [The Cardinality Explosion Problem](#14-the-cardinality-explosion-problem)
15. [Percentile Aggregation Correctness](#15-percentile-aggregation-correctness)
16. [CAP / PACELC Positioning](#16-cap--pacelc-positioning)
17. [Failure Modes & Mitigations](#17-failure-modes--mitigations)
18. [Scalability & Sharding](#18-scalability--sharding)
19. [Retention & Downsampling Strategy](#19-retention--downsampling-strategy)
20. [Senior vs Staff Answer Differentiators](#20-senior-vs-staff-answer-differentiators)
21. [Interview Time Allocation](#21-interview-time-allocation)
22. [Quick-Reference Cheatsheet](#22-quick-reference-cheatsheet)
23. [Related Guides](#23-related-guides)

---

## 1. Problem Statement & Scope

A **metrics monitoring platform** ingests numerical time-series telemetry (counters, gauges, histograms/distributions) from hosts, containers, and applications; stores it durably and cheaply at massive scale; supports flexible ad-hoc and dashboard queries with aggregation across time and tags; and evaluates alerting rules continuously to detect and notify on anomalies — often within seconds of the underlying condition occurring.

Scope this to **metrics only** early in the interview. Logs and distributed traces (APM) are adjacent systems with very different data shapes (unstructured text, DAGs) and typically get explicitly carved out — mention this as a scoping decision, not an oversight.

### What the interviewer is really testing

- Can you design an ingestion pipeline that survives **bursty, adversarial, multi-tenant write load** without ever silently dropping data — especially during the outage the metrics exist to detect?
- Do you understand **time-series data modeling** and, critically, the **cardinality explosion problem**?
- Can you design a query/aggregation engine where partial results from many shards are **mathematically mergeable** (this is where naive designs quietly produce wrong numbers)?
- Can you design **real-time alerting** as a streaming problem rather than a polling problem?
- Do you make **explicit CAP/PACELC decisions per component**, recognizing that this system has an unusual bias toward write-availability?

---

## 2. Requirements

### Functional Requirements

| Requirement | Notes |
|---|---|
| Ingest metrics from agents/SDKs (hosts, containers, serverless) | Counters, gauges, histograms/distributions |
| Support arbitrary tags/dimensions per metric | `env:prod, service:checkout, region:us-west-2` |
| Query API: aggregate over time + group by tags | `sum:requests{service:checkout} by {region}` |
| Dashboards | Compose multiple queries into visual widgets |
| Alerting | Threshold, change, and anomaly-based rules evaluated continuously |
| Multi-tenancy | Hard isolation of data, quota, and blast radius per customer |
| Retention tiers | Raw resolution short-term; rolled-up resolution long-term |

### Non-Functional Requirements

| Requirement | Target |
|---|---|
| Scale | ~50M active time series platform-wide; ~5M points/sec ingested |
| Write availability | Effectively 100% — dropping metrics during an incident defeats the product's purpose |
| Query latency | Interactive dashboards: p95 < 1–2s for common queries |
| Alert evaluation latency | Seconds, not minutes — outages must be detected fast |
| Durability | No data loss on single-node or single-AZ failure |
| Cost efficiency | Time-series volume is enormous; compression and downsampling are first-class cost levers, not afterthoughts |

---

## 3. Capacity Estimation

```
Assumptions:
  100,000 hosts monitored, ~500 distinct time series per host
    (system metrics + custom app metrics), 10s agent flush interval

Active time series:
  100,000 hosts × 500 series/host = 50,000,000 active series

Ingest rate:
  50,000,000 series / 10s interval ≈ 5,000,000 points/sec

Per-point size (compressed, Gorilla-style delta-of-delta + XOR):
  ~2–4 bytes/point for the (timestamp, value) pair once amortized
  over a compressed block; tag metadata stored once per series,
  not once per point.

Raw ingest bandwidth:
  5,000,000 pts/sec × ~16 bytes (wire format, pre-compaction)
    ≈ 80 MB/s

Daily raw volume (pre-replication):
  80 MB/s × 86,400s ≈ 6.9 TB/day
  × replication factor 3 ≈ ~20 TB/day durable

Kafka throughput:
  80 MB/s → ~16 partitions at 5 MB/s each (headroom for bursts → 32+)

Alert evaluation:
  1,000,000 active alert rules, evaluated every 10–60s
    ≈ 100,000 evaluations/sec at the low end

Tag index (metadata):
  50M series × ~200 bytes of tag metadata ≈ 10 GB — fits in memory
  per shard; cardinality is the real risk, not raw index size (see §14)
```

**Key insight:** unlike the web crawler (where storage volume is the dominant cost), here the dominant *risk* is cardinality, not raw point volume. A single misbehaving tag (e.g., a customer tagging metrics with `request_id`) can turn 50M series into 5 billion overnight — a compression-resistant, index-breaking failure mode that has to be designed against from day one.

---

## 4. High-Level Architecture

```
                    ┌───────────────────────────┐
                    │   Agents / SDKs            │
                    │  hosts · containers ·      │
                    │  serverless · StatsD/OTel   │
                    └─────────────┬──────────────┘
                                  │ push: batched, pre-aggregated,
                                  │ locally buffered on backpressure
                    ┌─────────────▼──────────────┐
                    │   Ingestion Gateway         │
                    │   AuthN/quota · validation  │
                    │   cardinality guardrails    │
                    └─────────────┬──────────────┘
                                  │ Kafka: shard by hash(series_id)
                    ┌─────────────▼──────────────┐
                    │  Kafka topic: raw-metrics   │
                    └──────┬───────────────┬──────┘
                           │               │
             ┌─────────────▼───┐   ┌───────▼─────────────┐
             │ Stream Rollup    │   │  Alert Evaluation    │
             │ Workers (Flink)  │   │  Workers              │
             │ multi-resolution │   │  subscribe to rollup  │
             │ downsampling     │   │  stream, not raw TSDB │
             └─────┬────────────┘   └───────┬───────────────┘
                   │                        │
        ┌──────────▼───────────┐   ┌────────▼─────────────┐
        │  Time-Series DB       │   │  Notification         │
        │  Hot tier (SSD)       │   │  Fan-out               │
        │  Cold tier (S3 blocks)│   │  PagerDuty/Slack/email │
        └──────────┬────────────┘   └────────────────────────┘
                   │
        ┌──────────▼────────────┐        ┌──────────────────────┐
        │  Metadata/Tag Index    │◀──────▶│  Query Engine          │
        │  roaring-bitmap        │        │  parse → resolve tags  │
        │  postings per tag      │        │  → fan-out → merge      │
        └─────────────────────────┘        └───────────┬────────────┘
                                                         │
                                              ┌──────────▼──────────┐
                                              │  Dashboard Service   │
                                              └──────────────────────┘
```

**Kafka** again plays the durable-decoupling role it plays in the crawler design (see [Related Guides](#23-related-guides)): it lets rollup computation, alerting, and storage scale and fail independently, and it gives you replay — you can recompute rollups from raw points if a downsampling bug ships.

---

## 5. Component 1: Agent / Collector

### What it does

Runs on the host/container/function, collects raw measurements, tags them, does **local pre-aggregation**, batches, compresses, and pushes to the ingestion gateway.

### Local pre-aggregation

Rather than shipping every raw increment of a counter, the agent aggregates locally over the flush interval (typically 10s) and ships one point per series per interval. This is the single biggest lever for controlling both network chatter and ingestion-tier load — it moves the first aggregation step to the edge, where it's free.

### Push vs. Pull — a tradeoff, not a default

| Model | Pros | Cons |
|---|---|---|
| **Push** (agent → gateway) | Works for ephemeral/serverless workloads with no scrape target; agent can pre-aggregate and shed load locally; server doesn't need service discovery | Ingestion tier must provision for worst-case burst; agent misconfiguration can flood the platform |
| **Pull** (server scrapes agent) | Server controls cadence and backpressure; natural for long-lived, discoverable targets (Prometheus model) | Doesn't work for short-lived functions that may not exist when the scrape fires; requires service discovery infrastructure |

**Staff-level answer:** default to push with client-side pre-aggregation, local disk-backed buffering, and a circuit breaker that drops the lowest-priority metrics before it drops everything — but support a pull-based federation path for on-prem/Kubernetes scrape targets where discovery is already solved. Don't present this as a binary choice; it's a workload-dependent decision.

### Local buffering & backpressure

The agent maintains a small disk-backed queue. On a network blip or a 429/503 from the gateway, it buffers and retries with exponential backoff rather than dropping — mirroring the crawler's fetcher retry logic, but the failure direction is reversed: here the *edge* protects itself from *transient* backend unavailability, because losing telemetry during an incident is the worst possible failure mode for this system.

---

## 6. Component 2: Ingestion Gateway

Stateless, horizontally scaled, sits behind a load balancer. Analogous role to the crawler's URL Router: normalize, validate, and route before anything durable happens.

```
Point from agent
      │
      ▼
┌─────────────────────────────────────────┐
│ Step 1: AuthN + tenant resolution        │
│  • API key → tenant_id                   │
└─────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────┐
│ Step 2: Quota & rate limiting            │
│  • per-tenant points/sec ceiling         │
│  • noisy-neighbor isolation              │
└─────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────┐
│ Step 3: Schema validation                │
│  • metric name format, tag key/value     │
│    length limits, timestamp sanity       │
│    (reject points too far in the future) │
└─────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────┐
│ Step 4: Cardinality guardrail check      │
│  • consult tag index cardinality counter │
│  • over threshold? sample / bucket the   │
│    offending tag into "other"            │
└─────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────┐
│ Step 5: Route to Kafka                   │
│  partition = hash(series_id) % N         │
│  (series_id = metric_name + sorted tags) │
└─────────────────────────────────────────┘
```

**Why hash the full series_id, not just the metric name?** Downstream, a single stream-aggregation worker needs to own the complete windowed state for one series to compute a correct rollup without cross-node coordination — the same domain-locality argument the crawler makes for per-domain routing. Hashing only the metric name would fan a hot metric's millions of tag combinations across a handful of partitions unevenly; hashing the full series_id spreads load evenly and keeps aggregation state local.

---

## 7. Component 3: Message Bus (Kafka)

Durable buffer between ingestion and every downstream consumer (rollup workers, alert evaluators, optionally a raw-archive sink). Provides:

- **Replay** — recompute rollups from raw points after a downsampling bug fix, without re-ingesting from agents.
- **Backpressure isolation** — a slow TSDB write path doesn't back up into the ingestion gateway; consumer lag is the pressure valve.
- **At-least-once delivery** — offsets committed only after a rollup worker's window state is checkpointed, matching the crawler's "commit only after successful downstream push" pattern.

See the **Kafka guide** ([Related Guides](#23-related-guides)) for partition-count sizing, `acks`/`min.insync.replicas` durability contracts, and consumer-group rebalancing — not re-derived here.

---

## 8. Component 4: Stream Aggregation & Rollup Pipeline

### What it does

Consumes raw points per partition and maintains **per-series windowed state**, emitting pre-aggregated rollups at multiple resolutions: `10s (raw) → 1min → 5min → 1hr → 1day`.

### Aggregation function depends on metric type

| Metric type | Rollup function |
|---|---|
| Counter | Sum over window |
| Gauge | Last, avg, min, max over window (configurable) |
| Histogram/Distribution | **Merge digests** (t-digest / DDSketch), not raw values — see [§15](#15-percentile-aggregation-correctness) |

### Watermarking and late data

Out-of-order or late-arriving points (clock skew, agent buffering after a network blip) are handled with a bounded lateness window (e.g., accept up to 5 minutes late). Points arriving later than that are either dropped or written to a side channel for manual backfill — an explicit, named tradeoff between correctness and unbounded memory growth in the stream processor's state store.

### Fault tolerance

Stream processors (Flink-style) checkpoint windowed state to durable storage (e.g., RocksDB state backend snapshotted to S3) on an interval. Kafka offsets commit only after a successful checkpoint, so a worker crash mid-window replays from the last checkpoint rather than losing partial aggregates — directly analogous to the crawler's "commit offset only after successful push to parser queue."

---

## 9. Component 5: Time-Series Storage Engine

### Data model

```
series_id = hash(metric_name + sorted(tag_key=value, ...))

Point = (timestamp, value)

Storage unit = compressed block of points for one series
               over one time chunk (e.g., 2 hours)
```

### Compression

Gorilla-style encoding (Facebook's time-series compression paper):
- **Timestamps:** delta-of-delta encoding — most intervals are constant (the flush interval), so the delta-of-delta is usually 0 and encodes in 1 bit.
- **Values:** XOR the current float against the previous float; leading/trailing zero runs compress the common case of slowly-changing metrics to a handful of bits.
- Typical result: **~1.3–2 bytes/point**, versus 16 bytes uncompressed.

### Two-tier storage (hot/cold), same pattern as the crawler's frontier

| Tier | System | Contents | Why |
|---|---|---|---|
| Hot | Local SSD-backed nodes | Recent raw + first-level rollups (last few hours to days) | Sub-ms reads for live dashboards |
| Cold | Object storage (S3/GCS), immutable blocks | Rolled-up, older data | Cheap, durable; matches Thanos/Cortex "block storage" pattern |

### Sharding is two-dimensional

1. **Series dimension:** consistent hashing on `series_id` distributes series evenly across storage nodes.
2. **Time dimension:** within a node, data is organized into **immutable, time-bucketed chunks** (e.g., 2-hour blocks). This makes retention trivial and cheap: expiring old data means deleting whole block files, never row-by-row deletes — the same LSM-style bulk-expiry trick the crawler guide uses for raw HTML retention, applied along the time axis instead of the crawl-date axis.

---

## 10. Component 6: Metadata / Tag Index Service

### What it resolves

A query like `sum:requests{env:prod, service:checkout}` must first resolve to a **concrete set of series_ids** before the TSDB can be queried — this is the inverted index that makes tag-based filtering possible.

```
Index structure: tag_key=value → roaring bitmap of series_ids

Query resolution:
  {env:prod, service:checkout}
    → bitmap(env=prod) AND bitmap(service=checkout)
    → intersection via roaring-bitmap AND (fast, SIMD-friendly)
    → resulting series_id set fed to Query Engine fan-out
```

**Why roaring bitmaps?** Tag-filter resolution is fundamentally set intersection across potentially millions of series. Roaring bitmaps give compressed, SIMD-friendly set operations that are dramatically faster than naive hash-set intersection at this scale — the metrics-platform equivalent of the Bloom filter's role in the crawler's dedup layer.

### Cardinality accounting lives here

The tag index is also where **per-tenant active-series counts** are tracked, because it's the single place that already knows every distinct series a tenant has ever emitted. This counter feeds the cardinality guardrail enforced back at the ingestion gateway (§6) and is the detection mechanism for [§14](#14-the-cardinality-explosion-problem).

---

## 11. Component 7: Query Engine

### Pipeline

```
Query string (e.g., PromQL/Datadog-style query language)
      │
      ▼
1. Parse into AST (metric, tag filters, group-by, function, time range)
      │
      ▼
2. Resolve tag filters → series_id set (via Metadata/Tag Index)
      │
      ▼
3. Choose resolution: does a pre-computed rollup at the requested
   granularity already exist? (Almost always yes for dashboard
   queries — this is the entire point of §8.)
      │
      ▼
4. Fan out to the TSDB shard nodes owning the resolved series_ids,
   in parallel, with a per-shard timeout
      │
      ▼
5. Merge partial results
   ⚠ merges must be associative/commutative — see §15
      │
      ▼
6. Apply final function (rate, percentile-from-sketch, top-k, etc.)
      │
      ▼
7. Return; cache result keyed by (query, time range, resolution)
```

### Caching

Because dashboards are disproportionately viewed by many users against the same handful of default queries, a short-TTL result cache (keyed by query + time range + resolution) absorbs the majority of read traffic before it ever reaches the TSDB fan-out path — the query-engine analogue of the crawler's DNS cache: a cheap, local win that removes load from the expensive path.

---

## 12. Component 8: Alerting Engine

### Streaming, not polling

A naive design re-runs each alert's query against the TSDB on a timer. At a million active rules, that's a query-fan-out storm. The Staff-level design instead has alert evaluators **subscribe directly to the rollup stream** produced in §8 — the aggregate a rule needs is often already being computed for dashboards, so evaluation is a cheap stream-join, not a fresh query.

### State machine per alert

```
        breach                for-duration elapsed
  OK ───────────▶ Pending ───────────────────▶ Alert ──▶ notify
   ▲                                              │
   │                recovery condition met         │
   └──────────────────────────────────────────────┘

  No Data: a distinct fourth state, entered when the expected
  metric stream itself stops arriving — NOT the same as OK.
```

**Why "No Data" matters at Staff level:** if an agent dies or a network partition cuts off a fleet of hosts, a naive threshold rule (`cpu > 90%`) simply has nothing to evaluate and silently reports nothing — which looks identical to "everything is fine." The single most dangerous monitoring failure mode is mistaking silence for health. A Staff-level design treats absence of expected data as its own alertable condition.

### Hysteresis

Rules require the breach condition to hold for a minimum duration (`for: 5m`) before transitioning to `Alert`, and typically a stricter recovery threshold than the trigger threshold, to prevent flapping on noisy metrics near a boundary.

### Alert-state consistency (a deliberate CP exception)

"Has this alert already fired, so we don't double-page on-call at 3am" is state that must not be lost or duplicated even under partition — see [§16](#16-cap--pacelc-positioning).

Notification fan-out (PagerDuty, Slack, email, webhook, with retry and channel fallback) is the same problem solved in the **Notification System guide** — reused here rather than re-derived.

---

## 13. Component 9: Dashboarding & API Layer

Dashboards are JSON definitions (widget layout + one query per widget) stored in a metadata store (Postgres/DynamoDB). Rendering a dashboard is simply issuing each widget's query to the Query Engine in parallel and rendering the results — the dashboard service itself holds no time-series logic.

---

## 14. The Cardinality Explosion Problem

This is this design's equivalent of the crawler's **spider trap** — a domain-specific correctness trap that a Staff candidate raises unprompted.

### The problem

A time series is uniquely identified by `metric_name + full tag set`. If a developer tags a metric with a naturally high-cardinality dimension — `request_id`, `user_id`, `pod_id` in a churning autoscaled fleet — the number of *distinct series* for that one metric name can explode from dozens to **billions**, because every new tag value mints a brand-new series.

```
Example:
  http.request.duration{service:checkout, region:us-west-2}
    → fine: bounded, low-cardinality tags

  http.request.duration{service:checkout, request_id:8f3a...}
    → catastrophic: one new series per request, forever
```

The damage compounds across multiple systems at once:
- **Storage:** millions of tiny, mostly-single-point series instead of a few long, well-compressed series — compression ratios collapse because Gorilla-style encoding relies on long runs per series.
- **Tag index:** roaring-bitmap postings balloon; index memory pressure spikes.
- **Query engine:** a group-by that used to resolve to hundreds of series now resolves to millions, blowing the fan-out budget for a single dashboard widget.

### Mitigations

- **Per-tenant series-count quota**, enforced at the ingestion gateway using the live counter maintained by the tag index (§10 ↔ §6).
- **Tag-value cardinality limiter per metric**: once a tag key exceeds N distinct observed values for a given metric, new distinct values are bucketed into a synthetic `"other"` value rather than minting a new series indefinitely.
- **Proactive customer-facing alerting**: notify the tenant when a metric is approaching its cardinality budget, before enforcement kicks in — turns a silent data-quality failure into an actionable signal.
- **Ingest-time linting**: flag known high-entropy tag key names (`request_id`, `trace_id`, `session_id`) heuristically and warn before the problem occurs at all.

**Senior-level answer:** "we use a Bloom/roaring-bitmap tag index." **Staff-level answer:** "cardinality is the dominant failure mode of this entire system, here's the guardrail at ingestion, here's the accounting that feeds it, and here's how we degrade gracefully instead of falling over."

---

## 15. Percentile Aggregation Correctness

A second correctness trap, distinct from cardinality, that separates Senior from Staff answers.

### The trap

Percentiles are **not associative**. Given per-shard p95 values from three storage nodes, you cannot average them, take their max, or otherwise naively combine them into a correct global p95 — the "average of averages" or "percentile of percentiles" pattern is mathematically wrong and silently produces plausible-looking, incorrect numbers.

```
WRONG:
  shard_A.p95 = 120ms, shard_B.p95 = 80ms, shard_C.p95 = 200ms
  global_p95 ≠ avg(120, 80, 200)   ← this number means nothing

RIGHT:
  each shard emits a mergeable sketch (t-digest or DDSketch),
  not a scalar percentile
  global sketch = merge(sketch_A, sketch_B, sketch_C)
  global_p95 = query(global sketch, 0.95)
```

### Design implication

The **rollup pipeline (§8)** must carry mergeable sketches end-to-end — from the agent's local pre-aggregation, through every rollup resolution, to the query engine's final merge step. Sums, counts, min, and max are trivially associative and safe to merge naively; percentiles and other quantile-based statistics are not, and the data structure choice has to be made at the point of first aggregation, not patched on later.

This is the same class of "silent wrongness" the crawler guide warns about with Bloom filter false positives, but the failure mode here is worse: a wrong percentile doesn't error out, it just quietly lies to whoever is looking at the dashboard during an incident.

---

## 16. CAP / PACELC Positioning

| Component | CAP Choice | Reasoning |
|---|---|---|
| Ingestion Gateway → Kafka | **AP** | Never block or drop incoming telemetry; availability is the whole point of the product, especially mid-incident. |
| Stream rollup / TSDB writes | **AP** | Accept writes under partition; reconcile via replication; a duplicated or slightly-delayed point is far cheaper than a lost one. |
| Metadata / Tag Index (cardinality counters) | **AP**, but conservative | Approximate, eventually-consistent counts are fine for normal accounting; guardrail enforcement is deliberately conservative (fail toward rejecting/bucketing rather than letting cardinality run away). |
| Query Engine reads | **AP / EL** (PACELC) | Even absent a partition, latency is favored over strong consistency: dashboards read from local rollup caches and merge across shards rather than doing a distributed transaction. |
| **Alert state (has this rule already fired?)** | **CP** | Must not double-page or silently drop a real page. A named exception, backed by a consensus store or a strongly consistent DB. |
| Dashboard / alert-rule definitions (control plane) | **CP** | Configuration changes must be consistent — the same reasoning the crawler guide applies to shard-assignment coordination via ZooKeeper/etcd. |

Consistent with the project's recurring framing: this is an **AP/EL system with two named CP exceptions** (alert-firing state, control-plane configuration) — not a system where every component gets re-litigated from first principles.

---

## 17. Failure Modes & Mitigations

| Failure | Symptom | Detection | Mitigation |
|---|---|---|---|
| Ingestion burst (mass host reboot, alert storm) | Gateway/Kafka saturation | Ingest latency/error rate spike | Agent-side local buffering + backoff; gateway autoscaling; per-tenant rate limits to contain blast radius |
| Cardinality explosion | Tag index memory spike; compression collapse | Per-tenant series-count crosses threshold | Guardrails from §14: quota, "other" bucketing, proactive tenant alert |
| Hot Kafka partition (one tenant's key dominates) | Partition lag on one broker | Per-partition lag monitoring | Virtual-node consistent hashing; split hot series-shards, same pattern as the crawler's hot-domain sub-queue split |
| Clock skew across agents | Points appear "late" or out of order | Late-data rate in stream aggregation | Bounded lateness window; NTP-synced agents; server-side ingest timestamp as secondary reference |
| Stream worker crash mid-window | Partial aggregate lost | Checkpoint/offset mismatch | Periodic state checkpointing; commit Kafka offset only after checkpoint succeeds |
| TSDB node failure | Reduced replicas for affected series | Replica health check | Replication factor 3 across AZs; consistent-hash ring rebalance; read repair |
| Query fan-out hits a slow shard | Dashboard widget stalls | Per-shard timeout exceeded | Return partial results with a "degraded" flag rather than blocking the whole query |
| Alert flapping | Repeated fire/recover on a noisy metric | High state-transition rate per rule | Hysteresis: minimum time-in-state before transition; asymmetric trigger/recovery thresholds |
| Silent data gap mistaken for "healthy" | No alerts fire despite an outage | Expected-metric-absence detector | Explicit "No Data" state, distinct from OK, itself alertable |
| Naive percentile merge | Dashboard shows a plausible but wrong p95 | Cross-check against raw-data spot audits | Mergeable sketches (t-digest/DDSketch) end-to-end, never scalar percentile averaging |
| Compaction/retention backlog | Cold-tier writes fall behind | Compaction lag metric | Throttle ingestion or scale compaction workers — the TSDB analogue of Kafka consumer lag |
| Downstream notification channel outage | Pages never reach on-call | Delivery-failure rate on notification fan-out | Retry queue + fallback channel (see Notification System guide) |

---

## 18. Scalability & Sharding

### Series sharding (primary axis)

```
shard_id = consistent_hash(series_id) % num_shards

Each shard owns:
  • its slice of the series namespace
  • its local aggregation state (no cross-shard coordination
    needed to compute one series' rollup)
  • its own set of time-bucketed storage blocks
```

### Time-bucketing (secondary axis, within a shard)

Each shard's data is additionally partitioned into immutable time chunks (e.g., 2-hour blocks), enabling bulk expiry for retention without per-row deletes — see §9.

### Horizontal scaling plan

| Component | Scaling strategy |
|---|---|
| Ingestion Gateway | Stateless; add nodes behind the load balancer |
| Kafka | Add partitions/brokers; partition key = series-shard key |
| Stream rollup workers | Add nodes; stateless *between* windows, stateful *within* a partition (Flink task parallelism = partition count) |
| Metadata/Tag Index | Shard by tenant or hash(metric_name); roaring-bitmap postings distributed across nodes |
| TSDB | Add nodes; consistent-hash ring with virtual nodes; replication factor 3 |
| Query Engine | Stateless fan-out layer; scale with query QPS |
| Cold storage (S3/GCS) | Managed; scales automatically |

### Backpressure

```
Monitor: rollup_consumer_lag = latest_kafka_offset - committed_offset

If rollup_consumer_lag > threshold:
  → scale up rollup worker fleet
  → OR apply ingestion-side rate limiting as a last resort
     (never silently drop; degrade resolution before dropping —
      e.g., skip an intermediate rollup tier under sustained load)
```

---

## 19. Retention & Downsampling Strategy

Different metrics warrant different resolution lifetimes; a fixed retention policy either wastes storage on data nobody queries at full resolution or throws away detail that matters.

```
Resolution   → Typical retention
10s (raw)     → 1–3 days   (incident investigation needs fine detail,
                             but only for recent events)
1 min         → 7–15 days
5 min         → 30–45 days
1 hour        → 13 months  (year-over-year dashboards)
1 day         → indefinite (long-term capacity trend lines)
```

This mirrors the crawler's freshness/recrawl tradeoff in reverse: instead of deciding *how often to refetch*, the platform decides *how long to keep each resolution tier*, with the same underlying principle — spend storage budget where queries actually look, and let a tiered rollup pipeline (§8) make the expensive fine-grained data disposable once its window has passed.

---

## 20. Senior vs Staff Answer Differentiators

### Senior-level expectations

- Identify the major components (agent, ingestion, storage, query, alerting)
- Explain time-series compression at a high level
- Mention tags/dimensions and basic aggregation (sum, avg)
- Describe threshold-based alerting
- Provide basic capacity math

### Staff-level expectations (additional)

| Topic | What Staff-level adds |
|---|---|
| Cardinality | Proactively raise cardinality explosion as the system's central failure mode, with concrete guardrails (§14) |
| Percentiles | Identify that percentile merging is non-associative and requires mergeable sketches end-to-end (§15) |
| Push vs. pull | Articulate the tradeoff with reasoning tied to workload shape (serverless vs. long-lived hosts), not a bare preference |
| Alerting | Distinguish "No Data" from "OK" as a first-class state; treat streaming evaluation as a scaling requirement, not an implementation detail |
| Sharding | Two-dimensional sharding (series hash + time buckets) and why each axis exists |
| CAP | Explicit AP/EL default with two named CP exceptions (alert-firing state, control-plane config) — not a blanket "eventually consistent" |
| Retention | Multi-resolution retention as a deliberate cost lever, computed and justified, not asserted |
| Failure modes | Proactively enumerate hot partitions, cardinality spikes, flapping, and silent data gaps before being asked |

### The single most important Staff differentiator

**Treating cardinality as the organizing constraint of the entire design**, the way the crawler guide treats spider traps and politeness. Every component — ingestion guardrails, tag index accounting, storage compression, query fan-out budgets — exists partly in service of containing cardinality. A Staff answer keeps returning to this thread; a Senior answer mentions it once as a footnote, if at all.

---

## 21. Interview Time Allocation

For a 45-minute system design session:

| Phase | Time | Focus |
|---|---|---|
| Requirements & scope | 5 min | Confirm metrics-only scope; 1B+ series-scale target; write-availability priority |
| Capacity estimation | 5 min | Points/sec, compression ratio, storage volume, cardinality risk |
| High-level architecture | 5 min | Draw all components; name Kafka as the decoupling layer |
| Ingestion & agent design (deep dive) | 8 min | Push vs. pull, local pre-aggregation, cardinality guardrails |
| Storage & rollup pipeline | 10 min | Compression, two-tier hot/cold, series+time sharding, mergeable sketches |
| Query engine & alerting | 7 min | Fan-out/merge correctness, streaming evaluation, No-Data state |
| Cardinality deep dive | 5 min | The central Staff-level trap; guardrails end-to-end |
| Failure modes & CAP | 5 min | Hot partitions, flapping, alert-state consistency exception |

**What to cut if short on time:** dashboarding/API layer detail, retention-tier specifics. **Never cut:** cardinality, percentile-merge correctness, or the No-Data alerting distinction — these are the questions that separate Staff from Senior in this domain.

---

## 22. Quick-Reference Cheatsheet

```
KEY ALGORITHMS
──────────────
Compression:      Gorilla (delta-of-delta timestamps + XOR values) | ~1.3-2 bytes/point
Tag resolution:   Roaring bitmaps | tag_key=value → series_id set | fast SIMD intersection
Percentiles:      Mergeable sketches (t-digest / DDSketch) | NEVER average-of-averages
Sharding:         Consistent hash on full series_id (metric + sorted tags)
Retention:        Multi-resolution rollup | 10s → 1m → 5m → 1h → 1d, tiered lifetimes

KEY NUMBERS
───────────
50M active series   →  5M points/sec ingested (10s flush interval)
~16 bytes/point raw →  ~80 MB/s ingest bandwidth
Compressed:         ~6.9 TB/day raw → ~20 TB/day with 3x replication
1M active alert rules → ~100K evaluations/sec (streamed, not polled)

KEY SYSTEMS
───────────
Message bus:      Kafka (replay, per-series-shard partitioning, backpressure)
Storage engine:   LSM/block-based TSDB, hot (SSD) + cold (S3) tiers
Tag index:        Roaring-bitmap inverted index, sharded by tenant/metric
Stream processor: Flink-style, checkpointed windowed state
Coordination:     ZooKeeper/etcd for shard assignment (CP)
Alert state:      Strongly consistent store (CP exception)

CAP DECISIONS
─────────────
AP/EL:  Ingestion, TSDB writes, tag-index counts, query reads
CP:     Alert-firing state, control-plane config (dashboards/rules)

STAFF-LEVEL SIGNALS TO HIT
───────────────────────────
✓ Cardinality explosion named as the system's central failure mode
✓ Guardrails for cardinality at ingestion AND accounting at the tag index
✓ Percentile merging called out as non-associative; mergeable sketches end-to-end
✓ Push vs. pull articulated as a workload-dependent tradeoff
✓ "No Data" as a distinct, alertable state from "OK"
✓ Two-dimensional sharding: series hash + time buckets
✓ Streaming alert evaluation instead of per-rule polling
✓ Explicit CAP default (AP/EL) with two named CP exceptions
✓ Multi-resolution retention framed as a cost lever
✓ Proactive failure-mode enumeration (hot partitions, flapping, silent gaps)
```

---

## 23. Related Guides

- **Apache Kafka guide** — partition sizing, `acks`/`min.insync.replicas` durability contract, consumer-group rebalancing. Not re-derived here; the message-bus role in this design assumes that guide's internals.
- **Sharding Strategies guide** — consistent hashing and virtual-node rebalancing, referenced rather than re-explained in §18.
- **CAP/PACELC Theorem guide** — the general framework underlying §16.
- **Notification System guide** — alert delivery fan-out (PagerDuty/Slack/email, retry, channel fallback) referenced rather than re-derived in §12.
- **Endpoint Telemetry Pipeline guide** — covers agent-side collection in more depth for the endpoint/security-telemetry use case; this guide's Agent/Collector component (§5) overlaps in mechanism (local pre-aggregation, backpressure buffering) but this guide's center of gravity is storage, query, and alerting at the metrics-platform layer, not the endpoint-agent layer.
- **Leaderboard System guide** — shares this guide's core Staff-level framing that *the sharding strategy determines which query becomes an expensive fan-out*.
- **Web Crawler guide** — structurally the template for this guide; several patterns are directly reused: Kafka as durable decoupling layer, per-key routing for locality (domain → series_id), two-tier hot/cold storage, and "proactively surface failure modes" as the top Staff differentiator.

---

*Guide compiled for Senior & Staff-level system design interview preparation.*
*Topics: Metrics Monitoring, Time-Series Databases, Cardinality, Stream Aggregation, Alerting, CAP Theorem.*
