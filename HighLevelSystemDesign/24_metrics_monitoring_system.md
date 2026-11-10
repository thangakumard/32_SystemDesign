# Metrics Monitoring & Alerting Platform — Staff-Level System Design

> Datadog / Prometheus+Mimir / Grafana Cloud–style platform: ingest time-series metrics from **500K servers at 5M samples/sec**, store them, power dashboards, evaluate alert rules, and notify on-call engineers.
> All numbers are **illustrative** and use decimal (SI) units.

---

## Table of Contents

0. [Interview Question → Section Map](#0-interview-question--section-map)
1. [Functional Requirements](#1-functional-requirements)
2. [Non-Functional Requirements](#2-non-functional-requirements)
3. [Capacity Estimation](#3-capacity-estimation)
4. [Core Entities](#4-core-entities)
5. [API / System Interface](#5-api--system-interface) — *Q1*
6. [Data Flow](#6-data-flow)
7. [High-Level Design](#7-high-level-design)
   - 7.1 [FR1 — Ingest 5M samples/sec](#71-fr1--ingest-metrics-from-500k-servers-at-5m-samplessec) — *Q2*
   - 7.2 [FR2 — Query & visualize on dashboards](#72-fr2--query--visualize-metrics-on-dashboards) — *Q3*
   - 7.3 [FR3 — Define & evaluate alert rules](#73-fr3--define--evaluate-alert-rules) — *Q4*
   - 7.4 [FR4 — Notify when alerts fire](#74-fr4--notify-when-alerts-fire) — *Q5*
8. [Deep Dives (Non-Functional)](#8-deep-dives-non-functional)
   - 8.1 [Low-latency queries over weeks/months](#81-low-latency-dashboard-queries-over-weeksmonths) — *Q6*
   - 8.2 [Real-time alerting for critical alerts](#82-real-time-alerting-for-critical-alerts) — *Q7*
   - 8.3 [High availability — never go blind](#83-high-availability--never-go-blind-during-incidents) — *Q8*
   - 8.4 [Preventing cardinality explosion](#84-preventing-cardinality-explosion) — *Q9*
9. [Domain Correctness Traps](#9-domain-correctness-traps)
10. [CAP / PACELC per Component](#10-cap--pacelc-per-component)
11. [Failure Modes & Mitigations](#11-failure-modes--mitigations)
12. [Scalability & Sharding](#12-scalability--sharding)
13. [Senior vs. Staff Differentiators](#13-senior-vs-staff-differentiators)
14. [45-Minute Interview Allocation](#14-45-minute-interview-allocation)
15. [Quick-Reference Cheatsheet](#15-quick-reference-cheatsheet)
16. [Related Guides](#16-related-guides)

---

## 0. Interview Question → Section Map

| # | Question | Where answered |
|---|---|---|
| Q1 | What external APIs will the system expose? | §5 |
| Q2 | How do we ingest from 500K servers at 5M metrics/sec? | §7.1 (HLD FR1) |
| Q3 | How do users query and visualize metrics on dashboards? | §7.2 (HLD FR2) |
| Q4 | How do users define alert rules and have them evaluated? | §7.3 (HLD FR3) |
| Q5 | How do users receive notifications when alerts fire? | §7.4 (HLD FR4) |
| Q6 | Low-latency dashboard queries over weeks/months of data? | §8.1 (Deep Dive NFR) |
| Q7 | Reduce alert latency to real-time for critical alerts? | §8.2 (Deep Dive NFR) |
| Q8 | High availability so monitoring doesn't go blind in incidents? | §8.3 (Deep Dive NFR) |
| Q9 | Prevent cardinality explosion? | §8.4 (Deep Dive NFR) |

Every section marked **🎤 Talk track** is sized for a **90–120 second** spoken answer (~220–280 words). Detail below each talk track is backup for follow-ups.

---

## 1. Functional Requirements

**Core (in scope)**
1. **Ingest metrics** — agents on 500K servers send time-series samples (`name{labels} value timestamp`): counters, gauges, histograms.
2. **Query & visualize** — users build dashboards with panels backed by a query language (PromQL-like): aggregations, rates, percentiles, group-by over arbitrary time ranges (last 5 min → last 13 months).
3. **Define & evaluate alert rules** — threshold, rate-of-change, absence ("no data"), and composite rules with a `for` duration; evaluated continuously.
4. **Notify** — route firing/resolved alerts to channels (PagerDuty, Slack, email, SMS/voice, webhooks) with grouping, deduplication, silencing, and escalation.

**Supporting**
5. Multi-tenancy (teams/orgs) with quotas.
6. Metadata exploration: list metrics, label names, label values.
7. Retention and downsampling policies per tenant.
8. Silences/maintenance windows, alert acknowledgment.

**Out of scope** (call out explicitly): logs and traces (separate pipelines — see *Endpoint Telemetry Pipeline*), anomaly-detection ML, billing.

---

## 2. Non-Functional Requirements

| NFR | Target | Why it matters |
|---|---|---|
| **Write throughput** | 5M samples/s sustained, 2–3× burst headroom (~15M/s) | Incidents cause retries, restarts, and new series all at once |
| **Availability** | Ingest + alerting **99.99%**; dashboards 99.9% | Monitoring must be *more* available than what it monitors |
| **Alert latency** | Standard: < 1–2 min end-to-end; **Critical: < 10–15 s** (excluding user `for`) | Mean time to detect drives MTTR |
| **Query latency** | p99 < 1 s for last-1h dashboards; p99 < 3–5 s for 30-day ranges | Dashboards are the incident war-room UI |
| **Durability** | No loss of acknowledged samples (RPO ≈ 0 within region) | Gaps break rate() math and SLO reporting |
| **Freshness** | Sample queryable within ~5–10 s of arrival | "Is my fix working?" during incidents |
| **Scalability** | Horizontal: linear in servers, series, tenants | 500K servers today, 2M tomorrow |
| **Cardinality control** | Hard per-tenant limits; one tenant cannot degrade others | #1 real-world cause of TSDB outages |
| **Retention** | Raw 15 d, 1-min rollups 30 d, 1-h rollups 13 mo | Year-over-year capacity planning |
| **Isolation / independence** | Platform must not depend on infrastructure it monitors | Avoid "blind during outage" |

**Anchor constraint:** *Alerting availability trumps everything.* When resources are scarce, shed in this order: dashboards → long-range queries → ingestion of non-critical metrics → **never** alert evaluation/notification.

---

## 3. Capacity Estimation

**Assumptions (illustrative)**
- 500K servers, each emitting ~100 series at a 10-s interval → 500K × 100 / 10 = **5M samples/s**.
- Active series ≈ 500K × 100 = **50M**; with container churn, plan for **~100M active series/day**.

**Ingest bandwidth**
- Wire format: protobuf + snappy, batched (labels sent once per batch). ~20 B/sample compressed → **5M × 20 B ≈ 100 MB/s** cluster-wide (≈ 0.8 Gbps).
- Per server: 10 samples/s × 20 B = 200 B/s — trivial; the challenge is aggregate fan-in and connection count (500K long-lived connections → ~50 gateways at 10K conns each).

**Kafka**
- 100 MB/s in × RF3 = 300 MB/s replicated write. 24-h retention buffer ≈ 8.6 TB × 3 ≈ **26 TB**.
- Partitions: ~10 MB/s per partition comfortable → ≥ 10, plan **~256–512 partitions** for consumer parallelism and future growth.

**Storage (Gorilla-style compression: ~1.4 B/sample vs. 16 B raw)**
| Tier | Points/day | Bytes/point | Per day | Retention | Total (×1) |
|---|---|---|---|---|---|
| Raw 10 s | 5M × 86,400 = 432B | ~1.4 B | ~600 GB | 15 d | ~9 TB |
| 1-min rollup (min/max/sum/count) | 50M × 1,440 = 72B | ~6 B | ~430 GB | 30 d | ~13 TB |
| 1-h rollup | 50M × 24 = 1.2B | ~6 B | ~7 GB | 13 mo | ~3 TB |
| **Total** | | | | | **~25 TB** |

- Stored in object storage (S3/GCS), which provides its own durability; ingester WAL/local disks replicated RF3 for the ~2 h hot window.

**Index**
- 50–100M series × ~1 KB (labels + postings) ≈ **50–100 GB** inverted index — sharded across ingesters/store-gateways, mostly memory-resident or mmap'd.

**Ingester memory (hot head block, last ~2 h)**
- 50M series × (720 samples × 1.4 B + ~1 KB overhead) ≈ 50M × 2 KB ≈ **100 GB × RF3 = 300 GB** → ~50 ingesters at ~6–8 GB head each, plus headroom.

**Query load**
- 10K concurrent dashboard viewers × 20 panels / 30-s refresh ≈ **~7K queries/s** (client-facing); each fans out to N queriers/ingesters internally (internal fan-out ≫ 7K/s).

**Alert evaluation**
- 1M rules / 60-s interval ≈ **~17K rule evals/s**; critical subset (say 50K rules) evaluated on the stream.

---

## 4. Core Entities

| Entity | Key fields | Notes |
|---|---|---|
| **Tenant** | tenant_id, limits (max_series, ingest_rate, max_query_range), retention | Quotas enforced at gateway & ingesters |
| **Series** | series_id = hash(tenant_id, metric_name, sorted(labels)), labels map | Identity of a time series; labels define cardinality |
| **Sample** | (series_id, timestamp_ms, float64 value) | Natural idempotency key = (series_id, ts) |
| **Histogram sample** | series_id, ts, buckets / sketch | Native/exponential histograms or DDSketch for mergeable percentiles |
| **Chunk / Block** | block_id, tenant, min_ts, max_ts, chunks, index, resolution | Immutable 2-h blocks in object storage, compacted to 24 h |
| **Posting list** | (label_name, label_value) → sorted [series_id] | Inverted index for label matching |
| **Dashboard** | dashboard_id, tenant, panels[{query, viz, step}], variables, version | Stored in config DB (Postgres) |
| **AlertRule** | rule_id, tenant, expr, for, eval_interval, severity, labels, annotations, tier (standard/critical) | Config DB; versioned |
| **AlertInstance** | fingerprint = hash(rule_id, label set), state (inactive/pending/firing/resolved), active_since, last_eval | One rule can produce many instances (one per host) |
| **NotificationPolicy** | route tree: matchers → contact point, group_by, group_wait, repeat_interval, escalation | Alertmanager-style routing tree |
| **ContactPoint** | type (pagerduty/slack/email/sms/webhook), config, secrets ref | Secrets in vault, not config DB |
| **Silence** | matchers, starts_at, ends_at, created_by, comment | Suppresses matching notifications |
| **Notification** | notification_id, group_key, contact_point, payload, status, attempts | Delivery log + idempotency |

---

## 5. API / System Interface

### 🎤 Talk track — Q1: What external APIs will the system expose? (90–120 s)
- **Four API families, mapped to the four functional requirements**: ingest, query, alerting config, and notification/routing config — plus admin.
- **Ingest API — `POST /v1/write`**: batched, protobuf + snappy, compatible with Prometheus remote-write and OTLP so we don't force a proprietary agent. Authenticated with a per-tenant API key. Returns `204` on durable commit to Kafka, `429` with `Retry-After` on rate/cardinality limits.
- **Writes are naturally idempotent**: the key is `(series_id, timestamp)`, so agent retries after a timeout don't double-count — no separate idempotency token needed.
- **Query API — `GET /v1/query` and `/v1/query_range`**: PromQL-style expression, `start`, `end`, `step`. Plus metadata endpoints — `/v1/labels`, `/v1/label/{name}/values`, `/v1/series` — which power autocomplete in the dashboard editor.
- **Dashboard CRUD — `/v1/dashboards`**: versioned with optimistic concurrency (`If-Match` ETag) so two engineers editing during an incident don't clobber each other.
- **Alert rules — `/v1/alert-rules`** CRUD, plus **`POST /v1/alert-rules:test`** — a dry run that evaluates the rule over the last 24 h and shows when it *would* have fired. That's how users avoid noisy rules.
- **Active alerts — `GET /v1/alerts`**, **`POST /v1/alerts/{fingerprint}:ack`**.
- **Routing — `/v1/contact-points`, `/v1/notification-policies`, `/v1/silences`.**
- **Admin — `/v1/tenants/{id}/limits`, `/v1/cardinality`** to expose top-N series contributors.
- Users authenticate via OIDC; machines via scoped API keys. All config APIs are low-QPS and hit Postgres; only `/write` and `/query` are on the hot path.

### API details

```http
# ---------- Ingest (hot path, ~250K req/s if 20 samples/req; more realistically ~50K req/s with bigger batches) ----------
POST /v1/write
Content-Type: application/x-protobuf
Content-Encoding: snappy
X-Tenant-Key: <api-key>
Body: WriteRequest { timeseries: [ { labels:[{name,value}], samples:[{ts_ms, value}] , histograms:[...] } ] }
→ 204 No Content            (durably in Kafka: acks=all, min.insync.replicas=2)
→ 400 Bad Request           (malformed, too many labels, label too long)
→ 429 Too Many Requests     (rate or series limit) + Retry-After + reason: "series_limit"|"rate_limit"
→ 503 Service Unavailable   (agent buffers to disk and retries with jittered backoff)

# ---------- Query ----------
GET /v1/query?query=<expr>&time=<ts>
GET /v1/query_range?query=sum by (service)(rate(http_requests_total{env="prod"}[5m]))&start=..&end=..&step=60s
→ 200 { resultType: "matrix", result:[{metric:{...}, values:[[ts, "v"],...]}], stats:{resolution:"1m", cached:true} }

GET /v1/labels?match[]=...&start=..&end=..
GET /v1/label/{name}/values?match[]=...
GET /v1/series?match[]=...

# ---------- Dashboards ----------
POST   /v1/dashboards                  { title, panels:[{title, query, viz, step}], variables }
GET    /v1/dashboards/{id}
PUT    /v1/dashboards/{id}  If-Match: "v12"
DELETE /v1/dashboards/{id}

# ---------- Alert rules ----------
POST   /v1/alert-rules  { name, expr:"avg by(host)(cpu_usage) > 0.9", for:"5m", eval_interval:"30s",
                          severity:"page", tier:"critical"|"standard", labels:{team:"payments"},
                          annotations:{summary, runbook_url} }
GET    /v1/alert-rules?team=payments
PUT    /v1/alert-rules/{id}
DELETE /v1/alert-rules/{id}
POST   /v1/alert-rules:test  { expr, for, lookback:"24h" }  → [{fired_at, resolved_at, labels}]

GET    /v1/alerts?state=firing&severity=page
POST   /v1/alerts/{fingerprint}:ack     { user, comment }

# ---------- Notification routing ----------
POST /v1/contact-points        { type:"pagerduty", config:{integration_key_ref:"vault://..."} }
PUT  /v1/notification-policies { root:{ group_by:["alertname","cluster"], group_wait:"30s",
                                  group_interval:"5m", repeat_interval:"4h",
                                  routes:[{matchers:{severity:"page"}, contact:"pd-payments"}] } }
POST /v1/silences              { matchers:{cluster:"us-east-1a"}, starts_at, ends_at, comment }

# ---------- Admin ----------
GET  /v1/cardinality?tenant=..&top=20    → top metrics/labels by active series
PUT  /v1/tenants/{id}/limits             { max_active_series, ingest_rate, max_series_per_metric }
```

**Ingest protocol options**

| Option | Pros | Cons |
|---|---|---|
| HTTP/1.1 JSON | Universal, debuggable | 5–10× larger payloads, CPU-heavy parsing at 5M/s |
| **HTTP/2 + protobuf + snappy (remote-write / OTLP)** ✅ | Compact, ecosystem-compatible, multiplexed | Binary, needs tooling to debug |
| gRPC streaming | Lowest overhead, flow control | Long-lived streams complicate L7 load balancing and rebalancing |
| StatsD/UDP | Fire-and-forget, zero client blocking | Lossy, no auth, no backpressure |

---

## 6. Data Flow

```
WRITE PATH
 1. Agent collects (scrape local exporters / SDK push) → batches every 1–10 s
 2. Agent → LB → Ingest Gateway (auth, validate, rate limit, cardinality check)
 3. Gateway → Kafka topic metrics.raw (partition = hash(tenant, series_id)), acks=all
 4. Gateway → 204 to agent (sample is now durable)
 5. Ingesters consume their partitions → append to WAL → in-memory head block (queryable now)
 6. Every 2 h: head → immutable TSDB block → object storage; index updated
 7. Compactor merges 2 h → 24 h blocks; Downsampler emits 1-min & 1-h rollup blocks

READ PATH
 1. Browser → Dashboard Service (panel queries) → Query Frontend
 2. Frontend: auth, limits, align step, split by day, check Results Cache
 3. Cache misses → Query Scheduler → Queriers (parallel)
 4. Queriers fetch: recent (< 2–12 h) from Ingesters; historical from Store Gateways (object storage + index/chunk cache)
    → pick resolution tier by step (raw / 1m / 1h)
 5. Merge, dedup replicas, evaluate expression → frontend stitches splits → caches immutable parts → returns

ALERT PATH (standard)
 1. Rule Scheduler assigns rule groups to Rule Evaluators (sharded by consistent hash)
 2. Every eval_interval: evaluator runs query against Queriers (hot data from ingesters)
 3. State machine per AlertInstance: inactive → pending (for) → firing → resolved
 4. State changes → Alert Router (Alertmanager cluster)

ALERT PATH (critical / streaming)
 1. Stream Evaluator (Flink) consumes metrics.raw directly
 2. Pre-filtered by matchers of critical rules → windowed eval on arrival
 3. Firing → Alert Router via priority lane

NOTIFICATION PATH
 1. Alert Router: dedup by fingerprint → apply silences/inhibition → group (group_by, group_wait)
 2. Route tree → contact points → per-channel queues (Kafka topics: notify.page, notify.chat, notify.email)
 3. Channel workers deliver (PagerDuty / Slack / SMTP / Twilio / webhook) with retries + idempotency key
 4. Escalation scheduler: unacked after N min → next tier
 5. Delivery status → Notification log; acks flow back via API/webhooks
```

---

## 7. High-Level Design

### 7.0 Full architecture

```
                                     ┌──────────────────────────── CONTROL PLANE (Postgres, low QPS) ─────────────────────────────┐
                                     │  Config API ── Postgres (dashboards, rules, policies, silences, tenants/limits) ── Vault    │
                                     └──────────────┬───────────────────────────────┬───────────────────────────┬────────────────┘
                                                    │ rule sync                     │ limits (cached locally)   │ routing config
  ┌─────────────────┐                               ▼                               ▼                           ▼
  │ 500K servers    │   ┌─────────┐   ┌───────────────────┐   ┌──────────────────┐   ┌───────────────────────────────────────┐
  │ Agent (OTel/    │──▶│ GeoDNS/ │──▶│ Ingest Gateways   │──▶│ Kafka            │──▶│ Ingesters (sharded by partition,      │
  │ Prom agent)     │   │ L7 LB   │   │ stateless, ~50    │   │ metrics.raw      │   │ 2 replicas/partition across AZs)      │
  │ disk buffer 2h  │   └─────────┘   │ auth·validate·    │   │ 512 partitions   │   │ WAL + in-memory head (last ~2–12 h)   │
  └─────────────────┘                 │ rate·cardinality  │   │ RF3, acks=all    │   └──────────────┬───────────┬────────────┘
                                      └───────────────────┘   └───────┬──────────┘                  │ flush 2h  │ recent reads
                                                                      │                             ▼ blocks    │
                                              ┌───────────────────────┘                  ┌─────────────────┐    │
                                              ▼ (critical rules only)                    │ Object Storage  │    │
                                    ┌───────────────────┐                                │ (S3) TSDB blocks│◀── Compactor +
                                    │ Stream Evaluator  │                                │ raw/1m/1h tiers │    Downsampler
                                    │ (Flink, windowed) │                                └───────┬─────────┘    │
                                    └────────┬──────────┘                                        │              │
                                             │ priority lane                         ┌───────────▼─────────┐    │
                                             │                                       │ Store Gateways      │    │
                                             │                                       │ (index/chunk cache, │    │
                                             │                                       │  memcached)         │    │
                                             │                                       └───────────┬─────────┘    │
                                             │                                                   │              │
  ┌─────────────┐   ┌────────────────┐   ┌───▼───────────────┐   ┌──────────────┐   ┌────────────▼──────────────▼─┐
  │ Browser /   │──▶│ Dashboard Svc  │──▶│ Query Frontend    │──▶│ Query        │──▶│ Queriers (stateless, fan-out│
  │ Grafana UI  │   │ (panels→query) │   │ split·align·cache │   │ Scheduler    │   │ merge + dedup replicas)     │
  └─────────────┘   └────────────────┘   │ Results Cache     │   │ (fair queue  │   └──────────────▲──────────────┘
                                         └───────────────────┘   │  per tenant) │                  │
                                                                 └──────────────┘                  │ queries every eval_interval
                                                                                         ┌─────────┴───────────┐
                                                                                         │ Rule Evaluators     │
                                                                                         │ (sharded rule groups│
                                                                                         │  + alert state)     │
                                                                                         └─────────┬───────────┘
                                                                                                   │ state changes
                                                                      ┌────────────────────────────▼───────────────────────────┐
                                                                      │ Alert Router cluster (Alertmanager-style, 3 nodes,     │
                                                                      │ gossip): dedup · silence · inhibit · group · route     │
                                                                      └──────┬─────────────────┬────────────────┬──────────────┘
                                                                             ▼                 ▼                ▼
                                                                      notify.page        notify.chat      notify.email   (Kafka)
                                                                             │                 │                │
                                                                      PagerDuty/Twilio     Slack/Teams       SMTP/SES  ← channel workers
                                                                      (2 vendors, CB)                                     + escalation

   META-MONITORING: separate small stack in another region/provider + external dead-man's-switch heartbeat
```

**Per-component "why"**

| Component | Why it exists |
|---|---|
| **Agent with disk buffer** | Batches and compresses at the edge; buffers 1–2 h during backend outages so we don't lose data or overload on recovery |
| **Ingest Gateway (stateless)** | Single choke point for auth, validation, rate limits, and cardinality limits — protects everything downstream |
| **Kafka** | Decouples bursty writers from storage; replay on ingester crash; fan-out to multiple consumers (ingesters, stream evaluator, future consumers) |
| **Ingesters** | Build compressed in-memory head blocks; serve fresh data (most queries and all alert evals hit the last 2 h) |
| **Object storage** | Cheap, 11-nines durable, infinitely scalable long-term store; immutable blocks make caching trivial |
| **Compactor/Downsampler** | Fewer, larger blocks = fewer index lookups; rollups make long-range queries cheap |
| **Store Gateway** | Makes object storage queryable: caches block indexes and hot chunks |
| **Query Frontend** | Splits long queries, aligns steps, caches results, protects queriers from query-of-death |
| **Query Scheduler** | Per-tenant fair queuing so one tenant's heavy query doesn't starve others |
| **Rule Evaluators** | Periodic alert evaluation, sharded; keep alert state machine |
| **Stream Evaluator** | Sub-10-s alerting for a small set of critical rules |
| **Alert Router** | Turns thousands of raw alert instances into a handful of actionable notifications |
| **Per-channel queues + workers** | Isolate slow channels (email) from urgent ones (page); retries and vendor failover |
| **Meta-monitoring** | Something must watch the watcher, in a separate failure domain |

### Storage engine options (the biggest design decision)

| Option | Pros | Cons | Verdict |
|---|---|---|---|
| **Postgres / TimescaleDB** | SQL, familiar, joins with metadata | Write amplification and per-row overhead at 5M/s; sharding is manual; poor compression vs. Gorilla | ❌ Fine at 50K/s, not 5M/s |
| **Cassandra / Scylla wide rows** (`(series_id, day)` partition, ts clustering) | Proven write scale, tunable consistency, multi-DC | No native label index (need separate index); compaction tuning pain; ~10× worse compression than TSDB chunks; expensive at scale | ⚠️ Workable (early OpenTSDB/KairosDB pattern) |
| **Single-node Prometheus TSDB** | Excellent compression, simple | Not horizontally scalable, no HA, local disk retention | ❌ As a building block only |
| **Horizontally scaled TSDB on object storage (Mimir/Cortex/Thanos/M3 pattern)** ✅ | Gorilla compression (~1.4 B/sample), inverted label index, cheap S3 long-term, horizontally scalable read & write paths | Many moving parts; ingester state management; query fan-out complexity | ✅ **Chosen** |
| **Columnar OLAP (ClickHouse / Druid / Pinot)** | Great for high-cardinality ad-hoc analytics, SQL | Weaker at PromQL-style per-series rate()/staleness semantics; merges under heavy small inserts need batching | ⚠️ Good complement for high-cardinality "events" analytics |
| **Elasticsearch** | Flexible querying | Poor storage efficiency for numeric time series, heavy JVM ops | ❌ |

---

### 7.1 FR1 — Ingest metrics from 500K servers at 5M samples/sec

#### 🎤 Talk track — Q2 (90–120 s)
- **Math first**: 500K servers × ~100 series × 1 sample/10 s = 5M samples/s, ~50M active series. Compressed protobuf is ~20 B/sample, so only ~100 MB/s — the hard part is **fan-in and per-series state**, not bandwidth.
- **Push model with a local agent.** The agent scrapes local exporters, batches 1–10 s of samples, compresses with snappy, and sends via remote-write/OTLP. Pull from a central scraper doesn't scale cleanly to 500K targets across networks and NAT; push also handles short-lived jobs.
- **Agent buffers to disk for ~2 hours** and retries with jittered exponential backoff — so a backend blip doesn't lose data or cause a thundering herd on recovery.
- **Stateless ingest gateways** behind an L7 load balancer — ~50 nodes at 10K connections each. They authenticate, validate label sets, enforce per-tenant rate and **cardinality limits**, then write to Kafka.
- **Kafka is the durability boundary**: partition by `hash(tenant, series_id)` so all samples of a series land in order in one partition. `acks=all` with `min.insync.replicas=2`, RF3 — the gateway returns 204 only after that commit. Durability is that combined contract, not `acks` alone.
- **Ingesters** consume partitions, write a WAL, and append to an in-memory **head block** using Gorilla compression — delta-of-delta timestamps, XOR'd values — ~1.4 bytes/sample. Data is queryable within seconds.
- Every 2 hours the head is cut into an **immutable block** and uploaded to S3; a compactor merges and downsamples.
- **Writes are idempotent** on `(series_id, ts)`, so retries and Kafka replays are safe.
- **Scaling knob**: add partitions + ingesters; gateways scale on CPU.

#### Design detail

```
Agent ──batch(1–10s, snappy)──▶ LB ──▶ Gateway
                                        │ 1. auth (API key → tenant, cached)
                                        │ 2. validate (≤ 30 labels, ≤ 1KB per label set, ts within [-1h, +10m])
                                        │ 3. token-bucket rate limit per tenant
                                        │ 4. cardinality check (new series? → tenant active-series counter)
                                        ▼
                                  Kafka metrics.raw  (key = tenant|series_id)
                                        │
                     ┌──────────────────┼──────────────────┐
                     ▼                  ▼                  ▼
              Ingester P0–P63     Ingester P64–127   ...  (2 replicas per partition, different AZs)
              WAL → head block (per-series chunk: 120 samples or 2 h)
                     │
                     └─ every 2 h → block {chunks, index, meta.json} → S3
```

**Push vs. pull**

| | Pull (Prometheus scrape) | Push (agent → backend) ✅ |
|---|---|---|
| Pros | Target health = scrape success (free "up" metric); server controls rate; easy to debug | Works across NAT/firewalls; short-lived jobs; edge batching; scales to 500K hosts without central scrape coordination |
| Cons | Central scrapers must discover and reach 500K targets; NAT/cross-network pain; short-lived jobs missed | Need agent-side buffering & backpressure; liveness requires explicit heartbeat (`absent()` alerts) |
| **Hybrid (chosen)** | Agent **pulls locally** (scrapes localhost exporters) and **pushes remotely** — best of both | |

**Kafka in the middle vs. direct write to ingesters**

| | Direct gateway → ingesters (classic Cortex, RF3 quorum W=2) | Via Kafka ✅ |
|---|---|---|
| Pros | Lower latency, fewer components | Absorbs bursts; replay after ingester crash; multiple consumers (stream alerts, analytics); ingesters become simpler (Kafka owns replication) |
| Cons | Ingester outage → write errors; burst directly hits stateful tier; replication logic in ingesters | Extra hop (~5–50 ms); Kafka ops; partition count bounds parallelism |

> Note the two different "quorums": Kafka ISR (`min.insync.replicas`) and ingester `W+R>N` replication are **replication quorums**; Kafka controller (KRaft) and ring/lease ownership are **consensus quorums**. Don't conflate them.

---

### 7.2 FR2 — Query & visualize metrics on dashboards

#### 🎤 Talk track — Q3 (90–120 s)
- **User flow**: open a dashboard → the dashboard service loads its definition from Postgres — panels, queries, template variables — and the browser issues one `query_range` per panel.
- **The query frontend is the shield**: it authenticates, enforces limits like max range and max series, **aligns `start`/`end` to the step** so identical refreshes hit the same cache key, and **splits** a long query into day-sized sub-queries executed in parallel.
- **Results cache** (memcached): sub-results for completed days are immutable, so they're cached indefinitely; only the latest window is recomputed. A 7-day dashboard refresh typically recomputes just the last few minutes.
- **Query scheduler** keeps **per-tenant fair queues**, so one team's 90-day regex query can't starve everyone else.
- **Queriers are stateless**. They resolve label matchers via the inverted index into series IDs, then fetch chunks: the **last ~2–12 hours from ingesters** (in memory), older data from **store gateways** reading S3 blocks with index and chunk caches.
- **Resolution is chosen by step**: a 30-day panel on a 1,000-pixel chart needs ~1,000 points, so it reads 1-hour rollups, not 30 days of 10-second raw samples.
- Replicas are deduplicated, the PromQL engine evaluates `rate`, `sum by`, `histogram_quantile`, and the frontend stitches splits back together.
- **Incident thundering herd**: 200 engineers open the same dashboard at once → **request coalescing** (single-flight) plus the results cache means the backend computes it once.
- Exploration UX — autocomplete — uses the `/labels` and `/label/{name}/values` endpoints backed by the index.

#### Design detail

```
Browser ─▶ Dashboard Svc ─▶ Query Frontend
                               │ align step: start = floor(start/step)*step
                               │ split: [D-7 … D-1] (cacheable) + [today] (fresh)
                               │ cache lookup key = hash(tenant, expr, step, day)
                               ▼
                        Query Scheduler (per-tenant queues, max in-flight per tenant)
                               ▼
                        Queriers ──▶ Ingesters      (ts > now − 12h)
                               └──▶ Store Gateways (ts ≤ now − 2h) → index cache → chunk cache → S3
                                    overlap window deduped
```

**Query engine options**

| Option | Pros | Cons |
|---|---|---|
| PromQL (chosen) | Industry standard, great for rates/aggregations, ecosystem (Grafana) | Weak at joins and ad-hoc high-cardinality analytics |
| SQL (Timescale/ClickHouse) | Familiar, joins | Awkward rate()/counter reset/staleness semantics |
| Custom DSL | Tailored | Learning curve, no ecosystem |

**Freshness vs. cache**: never cache the most recent ~10 min (late/out-of-order samples may still arrive); mark sub-results "final" only once the out-of-order window has closed.

---

### 7.3 FR3 — Define & evaluate alert rules

#### 🎤 Talk track — Q4 (90–120 s)
- **Definition**: a user writes a rule — expression like `avg by (host)(cpu) > 0.9`, a `for` duration, severity, labels like `team=payments`, and a runbook link — via UI or rules-as-code in Git, synced through the config API into **Postgres**.
- **Dry-run before save**: `alert-rules:test` replays the rule over the last 24 h and shows when it would have fired. That's the single best tool against alert fatigue.
- **Rules are grouped and sharded**: ~1M rules → ~17K evals/s. A rule scheduler assigns rule groups to **rule evaluators** by consistent hashing, so adding evaluators rebalances only a slice.
- Each evaluator, every `eval_interval` (say 30–60 s), runs the query against queriers — which hit **ingesters' in-memory data**, never cold storage.
- **One rule yields many alert instances** — one per label set, such as per host. Each instance has a **fingerprint** = hash(rule, labels) and a state machine: inactive → **pending** → (condition true for `for`) → **firing** → **resolved**.
- **`for` suppresses flapping**; we also support hysteresis — fire at 90%, resolve below 85%.
- **State survives evaluator restarts**: persisted to a small state store, or reconstructed by re-evaluating the lookback window — otherwise every deploy resets `for` timers or re-fires.
- **Failure bias**: if shard ownership is ambiguous during rebalance, **two evaluators may both evaluate — that's fine**, the router dedups by fingerprint. A *missed* evaluation is the real failure.
- **"No data" is its own rule type**: `absent()` — because a dead host stops reporting rather than reporting zero.
- Firing and resolved transitions are sent to the alert router, re-sent periodically while firing so router state self-heals.

#### Design detail

```
          Git / UI ─▶ Config API ─▶ Postgres (rules, versioned)
                                      │ change stream
                                      ▼
                             Rule Scheduler (ring membership via etcd/gossip)
                    ┌─────────────────┼──────────────────┐
                    ▼                 ▼                  ▼
             Evaluator A        Evaluator B        Evaluator C     (rule groups hashed to owners)
               │ every 30s: query → Queriers → Ingesters (hot data)
               │ per-instance state:  inactive ─▶ pending ─(for elapsed)─▶ firing ─▶ resolved
               ▼
           Alert Router  (send on change + resend every 1m while firing)
```

**Alert evaluation options**

| Option | Pros | Cons | Use |
|---|---|---|---|
| **Periodic query-based** (Prometheus ruler) ✅ default | Full PromQL power; simple; reuses read path | Latency ≥ eval_interval + ingest lag; load on queriers | 95% of rules |
| **Streaming on ingest** (Flink/Kafka Streams) | Sub-second detection; no query load | Limited expressiveness; windowed state to manage; harder to evolve | Critical tier (§8.2) |
| **Agent-side / edge evaluation** | Fastest, works during backend partition | No global aggregations; config distribution to 500K hosts | Host-local heartbeats only |
| **Recording rules + simple threshold** | Precomputes expensive aggregations, cheap alerts | Extra series (cardinality) | Expensive expressions |

---

### 7.4 FR4 — Notify when alerts fire

#### 🎤 Talk track — Q5 (90–120 s)
- **Problem**: one network partition can produce **10,000 alert instances** in a minute. Paging 10,000 times is a failure. The notification layer's job is to turn alerts into a few **actionable** notifications.
- **Alert router cluster** — Alertmanager-style, 3 nodes across AZs, gossiping state. Every evaluator sends every alert to **all** router nodes; that redundancy is deliberate.
- **Pipeline**: **dedup** by fingerprint → apply **silences** for maintenance windows → **inhibition**, e.g. "cluster down" suppresses its 500 "host down" children → **grouping** by `alertname, cluster` with a `group_wait` of ~30 s so one page summarizes many instances.
- **Routing tree** matches labels to **contact points**: `severity=page, team=payments` → PagerDuty; warnings → Slack; info → email digest.
- **Per-channel Kafka queues** — `notify.page`, `notify.chat`, `notify.email` — so a slow SMTP relay never delays a page. Channel workers retry with backoff and send an **idempotency key** = hash(group_key, firing set, timestamp bucket), so retries don't double-page.
- **Vendor redundancy**: pages go to PagerDuty, with a circuit breaker failing over to a second SMS/voice vendor.
- **Escalation**: unacknowledged after 5 minutes → secondary on-call → manager. Acks come back via API or vendor webhook.
- **Resolved notifications** and **repeat_interval** reminders keep the incident channel honest.
- **Bias**: router nodes deliberately prefer **duplicate notifications over missed ones** — gossip is AP. A duplicate page is annoying; a missed page is an outage.

#### Design detail

```
Evaluators ──(send to all 3)──▶ Router-1 ◀─gossip─▶ Router-2 ◀─gossip─▶ Router-3
                                  │ dedup → silence → inhibit → group (group_wait 30s) → route
                                  │ only the node at position 0 for a group sends immediately;
                                  │ others wait N×peer_timeout and send only if not seen in gossip
                                  ▼
                 ┌───────────────┬───────────────┬────────────────┐
             notify.page     notify.chat     notify.email      notify.webhook   (Kafka)
                 ▼               ▼               ▼                 ▼
          PagerDuty ─CB─▶    Slack/Teams     SES/SMTP          customer URLs (signed, retried, DLQ)
          Twilio (backup)
                 │
          Escalation Scheduler (timer wheel / delayed queue): no ack in 5m → next tier
```

**Notification options**

| Option | Pros | Cons |
|---|---|---|
| Evaluator calls vendors directly | Simplest | No grouping/dedup/silences; slow vendor blocks evaluation |
| **Router + per-channel queues** ✅ | Grouping, inhibition, isolation, retries, audit log | More components |
| Fully managed (PagerDuty does routing) | Less to build | Vendor lock-in; still need dedup/grouping upstream |

---

## 8. Deep Dives (Non-Functional)

### 8.1 Low-latency dashboard queries over weeks/months

#### 🎤 Talk track — Q6 (90–120 s)
- **Root problem**: a 30-day query over 1,000 series at 10-s resolution is 260M samples. Nobody can render that, and no system should scan it per refresh. A chart is ~1,000 pixels wide — so we need ~1,000 points per series.
- **1. Downsampling / rollups**: the compactor produces **1-min and 1-hour** rollup blocks storing **min, max, sum, count** per window. The planner picks the coarsest tier whose resolution ≤ step. A 30-day query now reads ~720 points per series instead of 260K — a ~360× reduction.
- Store **sum and count, never avg**, so averages stay mathematically correct when re-aggregated; percentiles use **mergeable histograms or sketches**, never averaged p99s.
- **2. Query splitting + results cache**: split by day, run in parallel, and cache completed days forever because blocks are immutable. A refresh recomputes only the last few minutes.
- **3. Recording rules**: for hot, expensive queries — like `sum by (service)` over 50K pods — precompute the aggregate continuously as a new low-cardinality series. The dashboard reads 20 series, not 50K.
- **4. Storage-level caching**: store gateways cache block **indexes and postings** in memory and hot chunks in memcached; compaction into 24-hour blocks reduces how many indexes we touch.
- **5. Query sharding**: split by series hash so 16 queriers each aggregate a shard, then merge partial aggregates.
- **6. Guardrails**: max series per query, max samples, timeouts, per-tenant concurrency — a query of death gets rejected, not allowed to take down queriers.
- **Result**: p99 under ~3 s for 30-day panels, sub-second when cached.

#### Design detail

**Resolution selection**

| Query range | Step (≈1,000 pts) | Tier read | Samples/series |
|---|---|---|---|
| 1 h | 3.6 s → 10 s | raw (ingesters) | 360 |
| 24 h | 86 s → 1 m | 1-min rollup | 1,440 |
| 30 d | 43 min → 1 h | 1-h rollup | 720 |
| 13 mo | ~9 h → 1 h (re-aggregated) | 1-h rollup | ~9,500 → stepped |

**Options**

| Technique | Pros | Cons |
|---|---|---|
| Rollups/downsampling ✅ | Orders-of-magnitude fewer points | Loses fine detail; storage for extra tiers; must store correct aggregates |
| Results cache with splitting ✅ | Near-zero cost for repeat views | Only helps repeat queries; cache invalidation for late data |
| Recording rules ✅ | Precomputed hot aggregates | Must be anticipated; adds series |
| Query sharding ✅ | Parallelism for big aggregations | Fan-out cost; merge overhead |
| Columnar store (ClickHouse) for analytics | Fast scans for ad-hoc high-cardinality | Second system to operate |
| Bigger hardware only | Simple | Doesn't fix O(samples) scans |

---

### 8.2 Real-time alerting for critical alerts

#### 🎤 Talk track — Q7 (90–120 s)
- **Start with the latency budget** for the standard path: 10-s collection interval + ~2 s ingest + up to 60 s eval interval + `for` duration + ~5 s notification. **Detection alone is 15–75 s**, plus `for`. For critical alerts we want **under 10 seconds**.
- **Tiered alerting**: users mark a small set — say 50K rules — as `tier=critical`. Everything else stays on the periodic path. Don't make every alert real-time; it's expensive and most don't need it.
- **1. Shorten collection**: critical metrics are flushed by the agent every **1 s**, not 10 s.
- **2. Stream evaluation instead of polling**: a **Flink** job consumes `metrics.raw` directly from Kafka. A **matcher index** of critical rules filters samples at line rate, so only matching series enter keyed state, partitioned by series.
- Evaluation happens **on arrival** — sliding or tumbling windows for `rate` and `avg` — no query round-trip, no 60-s tick.
- **3. Replace `for: 5m` with count-based confirmation**: "3 consecutive breaches" or "80% of the last 15 s" — debouncing without minutes of delay.
- **4. Priority lane end-to-end**: dedicated Kafka topic with low `linger.ms`, a router path with `group_wait: 0` for critical pages, and pre-warmed PagerDuty connections.
- **5. Event-driven sources**: for binary signals — health-check failure, process crash — the agent emits an **event** immediately rather than waiting for a metric.
- **Trade-offs**: limited expression language on the streaming path (no arbitrary joins), more state to checkpoint, and higher false-positive risk — so we keep the periodic evaluator as a **backstop** for the same rule.
- **Result**: ~3–8 s detection-to-page.

#### Design detail

| Stage | Standard path | Critical path |
|---|---|---|
| Collection interval | 10 s | 1 s |
| Agent → Kafka | ~1–2 s | ~200 ms (linger 5 ms) |
| Evaluation trigger | every 30–60 s | on arrival |
| Debounce | `for: 1–5m` | N consecutive / % of window (10–15 s) |
| Router grouping | group_wait 30 s | group_wait 0 |
| Delivery | ~2–5 s | ~1–2 s |
| **Total (excl. debounce)** | **~45–75 s** | **~2–5 s** |

```
Kafka metrics.raw ─▶ Flink: [matcher filter] ─keyBy(series)─▶ [window state + rule eval] ─▶ alerts.critical
                                 ▲                                    │ checkpoints → S3
                       broadcast state: critical rules                ▼
                       (from config change stream)          Alert Router (priority path) ─▶ notify.page
```

| Option | Pros | Cons |
|---|---|---|
| Shorten eval_interval to 5 s on ruler | No new system | 12× query load; still bounded by ingest freshness |
| **Streaming evaluation (Flink)** ✅ | Sub-second detection, no query load | Limited expressiveness, stateful ops |
| Edge/agent evaluation | Fastest, partition-tolerant | Host-local only |
| Event-driven health checks | Instant for binary failures | Separate signal pipeline |

---

### 8.3 High availability — never go blind during incidents

#### 🎤 Talk track — Q8 (90–120 s)
- **Core insight**: monitoring load and importance **peak exactly when everything else is failing** — retries, restarts, dashboard herds. The design must assume correlated failure.
- **1. Independent failure domain**: the platform must not depend on what it monitors — its own Kafka, its own DNS path, its own compute pool. Otherwise the outage that breaks production also breaks visibility.
- **2. Redundancy at every tier, zone-aware**: stateless gateways and queriers across 3 AZs; Kafka RF3 spread across AZs; **two ingester replicas per partition** in different AZs; queriers dedup replica results. Losing an AZ loses capacity, not data or visibility.
- **3. Durable recovery**: ingester crash → replay from WAL and the last committed Kafka offset; agents buffer 2 hours to disk if the gateway tier is down.
- **4. Static stability**: gateways and evaluators **cache tenant limits, auth keys, and rules locally**. If Postgres or the control plane dies, ingestion and alerting keep running on last-known config.
- **5. Priority load shedding**: under pressure we shed dashboards first, then long-range queries, then non-critical ingestion — **never alert evaluation**. Alert evaluators have dedicated queriers and ingester read quotas.
- **6. Cell architecture**: regions are independent cells; critical alerts are evaluated in-region. Shuffle-sharding tenants across ingesters limits blast radius from a bad tenant.
- **7. Notification redundancy**: 3-node router cluster and two paging vendors behind a circuit breaker.
- **8. Watch the watcher**: a small meta-monitoring stack in a different region or provider, plus a **dead-man's-switch** — an always-firing heartbeat to an external service. If the heartbeat stops, *that* pages us.

#### Design detail

| Layer | Mechanism | Failure tolerated |
|---|---|---|
| Agent | 2 h disk buffer, jittered backoff | Full backend outage ≤ 2 h |
| Gateway | Stateless, N+2, multi-AZ | Node/AZ loss |
| Kafka | RF3, min.ISR=2, rack-aware | 1 broker or 1 AZ |
| Ingesters | 2 replicas/partition across AZs, WAL, Kafka replay | Node/AZ loss |
| Object storage | Provider 11 nines; optional cross-region replication | Region loss for history |
| Query path | Stateless queriers; serve partial results with warning header | Store-gateway subset loss |
| Rule evaluators | Consistent-hash ring; prefer double eval over missed eval | Node loss, rebalancing |
| Alert router | 3 nodes gossip; all evaluators send to all | 2 of 3 nodes |
| Notification | Per-channel queues, 2 paging vendors, CB | Vendor outage |
| Meta | Separate stack + external heartbeat | Whole primary platform |

---

### 8.4 Preventing cardinality explosion

#### 🎤 Talk track — Q9 (90–120 s)
- **What it is**: every unique label combination is a separate series with its own index entry, memory, and chunk. One engineer adding `user_id` or `request_id` as a label can turn 1 metric into **10 million series** — blowing ingester memory and index size and taking down the shared cluster for every tenant.
- **Defense in depth, from edge to storage**:
- **1. Prevent at the source**: agent-side relabeling rules drop known-bad labels — IDs, UUIDs, raw URLs — and normalize paths (`/users/123` → `/users/:id`). SDK linting in CI flags unbounded labels.
- **2. Enforce at the gateway**: per-tenant **active-series limit**, **per-metric series limit**, max labels per series, max label value length. New series beyond the limit get `429` with a clear reason; **existing series keep flowing** — we never blind a team's established metrics.
- Track per-metric cardinality with **HyperLogLog** sketches so the check is cheap at 5M samples/s.
- **3. Isolate at storage**: **shuffle sharding** puts each tenant on a subset of ingesters, so a cardinality bomb hits a few nodes, not all.
- **4. Detect early**: alert on series-creation rate and the top-N contributors; a cardinality explorer UI shows which metric and label exploded.
- **5. Offer a better home**: high-cardinality dimensions belong in **logs, traces, or exemplars** — exemplars attach a trace ID to a histogram bucket without creating series — or in a columnar store like ClickHouse for analytics.
- **6. Aggregate on ingest** as a fallback: drop the offending label and pre-aggregate — e.g. sum across `pod` — instead of rejecting.
- **Trade-off**: reject vs. drop-label vs. aggregate — rejecting is safe but loses data; we default to reject-new-series with a self-serve limit-raise workflow.

#### Design detail

```
Agent relabel (drop user_id, normalize paths)
   ▼
Gateway:
   series_id seen recently? (local bloom/LRU) ── yes ─▶ accept
          │ no (new series)
          ▼
   tenant.active_series < limit AND metric.cardinality(HLL) < per_metric_limit ?
          ├─ yes ─▶ accept, increment
          └─ no  ─▶ policy: reject(429) | drop_label | aggregate ; emit cardinality_limit_hit{tenant,metric}
   ▼
Ingesters: hard memory guard per tenant (last line of defense)
```

| Response to limit breach | Pros | Cons |
|---|---|---|
| **Reject new series** ✅ default | Protects platform; clear signal | Data for new series lost until fixed |
| Drop offending label | Keeps metric flowing | Silent semantic change; may merge distinct series (counter math breaks) |
| Aggregate on ingest | Keeps useful aggregate | Needs config; CPU on ingest path |
| Sample/throttle | Graceful | Gaps make rate() wrong |
| Unlimited + bill tenant | Simple product story | Still risks shared-cluster stability |

---

## 9. Domain Correctness Traps

These are the metrics-monitoring equivalent of the crawler's spider trap — a **pattern class** worth naming unprompted.

| Trap | What goes wrong | Mitigation |
|---|---|---|
| **Counter resets** | Process restart resets counter to 0 → naive delta is hugely negative | `rate()` detects decreases and treats them as resets; rollups store increase, not raw value |
| **Average of averages** | Rolling up averages gives wrong results with uneven counts | Store sum + count; compute avg = Σsum/Σcount |
| **Averaging percentiles** | avg(p99 per host) ≠ fleet p99 | Mergeable histograms / DDSketch / t-digest |
| **Absence ≠ zero** | Dead host stops reporting; threshold alerts never fire | `absent()` / staleness markers / heartbeat rules |
| **Out-of-order & late samples** | Agent buffer replay sends old data; breaks append-only chunks | Accept within an OOO window (e.g., 1 h); don't cache recent windows; compactor recomputes rollups |
| **Clock skew** | Agents with bad clocks write future/past timestamps | Reject ts outside [-1h, +10m]; optionally server-assign on ingest |
| **Alert storm** | One root cause → thousands of pages | Grouping, inhibition, dependency-aware routing |
| **Flapping** | Value oscillates around threshold | `for`, hysteresis, count-based confirmation |
| **Monitoring-on-same-infra** | Outage takes down monitoring | Independent failure domain + external dead-man's switch |
| **Cardinality bomb** | One label adds millions of series | §8.4 |
| **Query of death** | Regex over all series for 1 year | Limits, cost estimation, fair scheduling |

---

## 10. CAP / PACELC per Component

| Component | Position | Rationale |
|---|---|---|
| Ingest gateway / write path | **AP, PA/EL** | Accept writes during partitions; a slightly stale view is fine, rejecting writes is not |
| Kafka partitions | **CP-leaning, PC/EC** (acks=all, min.ISR=2) | Acked = durable; if ISR < 2, writes fail and **agents buffer** — availability recovered at the edge |
| Ingester replicas | **AP, PA/EL** | Two replicas consume independently; queriers merge/dedup; small divergence tolerated |
| Object storage blocks | **PC/EC** (immutable + strongly consistent reads) | Immutability removes consistency hazards |
| Results cache | **AP, EL** | Stale tolerated; recent window excluded |
| Config DB (rules, dashboards, policies) | **CP** (Postgres primary + sync replica) | Users must see what they saved; low QPS; consumers cache locally (static stability) |
| Rule evaluator ownership | **AP** | Prefer duplicate evaluation (deduped downstream) over a missed evaluation |
| Alert router state (dedup, silences) | **AP** (gossip) | Prefer duplicate notification over missed page; silences propagate eventually (seconds) |
| Tenant limits | **CP source, AP cache** | Authoritative in Postgres; gateways enforce cached values if control plane is down |

**Named CP exceptions**: Kafka durability contract, config DB, and the limits source of truth. Everything else is AP.

---

## 11. Failure Modes & Mitigations

| Failure | Impact | Mitigation |
|---|---|---|
| Ingester crash | Hot data on that node unavailable briefly | Replica in other AZ serves reads; restart replays WAL + Kafka from committed offset |
| Kafka broker loss | Partition leaders move | RF3, min.ISR=2; producers retry; agents buffer |
| AZ outage | 1/3 capacity lost | Zone-aware replication everywhere; N+1 capacity per AZ |
| Region outage | Region's cell down | Agents fail over to a secondary region endpoint; meta-monitoring in another region alerts |
| Cardinality bomb | Ingester OOM | Gateway limits, HLL tracking, shuffle sharding, per-tenant memory guard |
| Query of death | Querier OOM/CPU | Max series/samples, timeouts, fair queue, cost estimation |
| Dashboard thundering herd during incident | Query overload | Request coalescing (single-flight), results cache, step alignment |
| Alert storm (network partition) | Pager flood | Grouping, inhibition, cluster-level alerts suppress host-level |
| Paging vendor outage | Missed pages | Circuit breaker → secondary vendor; email/Slack fallback |
| Evaluator restart | `for` timers reset, re-fires | Persist alert state; restore from lookback |
| Compactor lag | Long-range queries slow, more blocks | Alert on compaction lag; scale compactors horizontally by tenant |
| Clock skew | Data rejected or misplaced | Timestamp bounds; NTP monitoring of agents |
| Control-plane (Postgres) outage | Can't edit rules/dashboards | Data plane runs on cached config (static stability) |
| Whole platform down | Blind | External dead-man's switch pages SRE |

---

## 12. Scalability & Sharding

**Write-path sharding**: `partition = hash(tenant_id, series_id) mod P`.
- Series affinity keeps per-series chunks on one ingester (compression needs sequential samples).
- **Trade-off**: sharding by series makes **write** perfectly balanced but makes every **query a fan-out** to all ingesters holding a tenant's series. Sharding by tenant would make queries local but creates hot tenants. → **Shuffle sharding**: each tenant maps to a subset of K ingesters (e.g., 8 of 200), hashed by series within that subset. Bounded fan-out + blast-radius isolation + balance.

**Read-path sharding**: queriers are stateless; store gateways shard blocks by `hash(block_id)`; query sharding splits by series hash for parallel aggregation.

**Rule evaluator sharding**: consistent hash of `(tenant, rule_group)`; ring membership via gossip/etcd.

**Hot spots**
- Hot tenant → shuffle shard size grows with tenant volume.
- Hot metric (e.g., `http_requests_total` with 2M series) → series-hash spreading inside the shard.
- Hot dashboard → results cache + coalescing.

**Growth to 4× (20M samples/s)**: add Kafka partitions (plan ahead — repartitioning breaks series ordering, so over-provision partitions early), add ingesters (ring rebalances a slice), add store gateways; object storage scales automatically.

---

## 13. Senior vs. Staff Differentiators

| Topic | Senior answer | Staff answer |
|---|---|---|
| Storage | "Use a time-series DB like InfluxDB" | Compares Postgres / Cassandra / TSDB-on-S3 / ClickHouse with compression math (16 B → 1.4 B/sample) and picks per access pattern |
| Ingestion | "Agents send to Kafka" | Push vs. pull hybrid, agent disk buffering, Kafka durability = `acks` + `min.ISR`, idempotency on (series, ts) |
| Long-range queries | "Add a cache" | Step-aligned splitting + immutable-block caching + rollups with correct aggregates (sum/count, mergeable sketches) |
| Alerting | "Cron job runs queries" | Latency budget breakdown, tiered streaming vs. periodic, `for` vs. count-based debounce, double-eval-over-missed-eval |
| Notification | "Send to PagerDuty" | Grouping, inhibition, silences, per-channel isolation, vendor failover, AP router that prefers duplicates |
| HA | "Replicate everything" | Independent failure domain, static stability, priority shedding (alerts never shed), dead-man's switch |
| Cardinality | "Limit labels" | Defense in depth: relabel → HLL limits → shuffle sharding → exemplars/logs; explains reject vs. drop vs. aggregate |
| CAP | System-wide "AP" | Per-component table with 3 named CP exceptions |
| Correctness | Not raised | Counter resets, absence ≠ zero, avg-of-avg, OOO samples raised unprompted |

---

## 14. 45-Minute Interview Allocation

| Time | Activity |
|---|---|
| 0–5 min | Functional & non-functional requirements; state anchor constraint ("alerting availability first") |
| 5–9 min | Capacity: 5M/s, 50M series, ~100 MB/s, 600 GB/day raw, ~25 TB tiered |
| 9–13 min | Core entities + APIs (write, query_range, alert-rules, policies) |
| 13–25 min | HLD diagram; walk write → read → alert → notify paths |
| 25–40 min | Deep dives: long-range queries, real-time alerts, HA, cardinality (follow interviewer's lead) |
| 40–45 min | Failure modes, CAP table, growth plan, correctness traps |

---

## 15. Quick-Reference Cheatsheet

```
NUMBERS      5M samples/s · 50M active series · ~100 MB/s wire · 1.4 B/sample (Gorilla)
             ~600 GB/day raw · ~25 TB tiered · ~7K dashboard qps · ~17K rule evals/s
WRITE        Agent(local pull, remote push, 2h disk buffer) → Gateway(auth/limits/cardinality)
             → Kafka(hash(tenant,series), RF3, acks=all, minISR=2) → Ingesters(WAL+head) → S3 blocks
READ         Frontend(align, split by day, results cache, coalesce) → Scheduler(fair queue)
             → Queriers → Ingesters (hot) + Store Gateways (S3, index/chunk cache)
ROLLUPS      raw 15d · 1m 30d · 1h 13mo · store min/max/sum/count · sketches for percentiles
ALERTS       periodic ruler (30–60s) for most · Flink streaming for critical (<10s)
             inactive→pending→firing→resolved · fingerprint dedup · double-eval > missed-eval
NOTIFY       dedup→silence→inhibit→group→route → per-channel Kafka queues → vendors(+backup, CB)
             escalation on no-ack · AP router prefers duplicates
HA           independent failure domain · zone-aware RF · static stability · shed dashboards first
             never shed alerting · dead-man's switch · cells + shuffle sharding
CARDINALITY  relabel at agent · HLL per-metric limits · reject NEW series only · shuffle sharding
             exemplars/logs/ClickHouse for high-card dimensions
TRAPS        counter resets · absence≠0 · avg-of-avg · p99 averaging · OOO samples · clock skew
CAP          AP everywhere except: Kafka durability, config DB, limits source of truth
```

---

## 16. Related Guides

- **Apache Kafka** — partitioning, `acks` + `min.insync.replicas`, consumer groups (ingest backbone here)
- **Endpoint Telemetry Pipeline** — agent design, edge buffering, high-volume ingestion patterns
- **Notification System** — channel workers, retries, idempotency, vendor failover (§7.4 reuses it)
- **Office 365 App Usage Tracking & Alert System** — threshold alerting, lazy vs. precomputed alert state
- **Sharding Strategies** — series-hash vs. tenant vs. shuffle sharding trade-off (§12)
- **CAP / PACELC Theorem** — per-component positioning (§10)
- **Distributed Key-Value Store (Dynamo-style)** — replication quorum `W+R>N` vs. consensus quorum
- **Leader Election in Distributed Systems** — ring/lease ownership for rule evaluators and compactors
- **Circuit Breaker Pattern** — paging-vendor failover
- **Leaderboard System** — "sharding strategy decides which query becomes the fan-out" (same lesson as §12)
