# Notification System — System Design Interview Guide
### Senior & Staff Engineer Level

---

## Table of Contents

1. [Problem Statement & Scope](#1-problem-statement--scope)
2. [Requirements](#2-requirements)
3. [Capacity Estimation](#3-capacity-estimation)
4. [High-Level Architecture](#4-high-level-architecture)
5. [Component 1: Notification Gateway (API)](#5-component-1-notification-gateway-api)
6. [Component 2: Preference & Subscription Service](#6-component-2-preference--subscription-service)
7. [Component 3: Template & Personalization Service](#7-component-3-template--personalization-service)
8. [Component 4: Router & Channel Selector](#8-component-4-router--channel-selector)
9. [Component 5: Message Queue / Event Bus](#9-component-5-message-queue--event-bus)
10. [Component 6: Channel Workers & Provider Integration](#10-component-6-channel-workers--provider-integration)
11. [Component 7: Device Token Management](#11-component-7-device-token-management)
12. [Component 8: Idempotency & Deduplication](#12-component-8-idempotency--deduplication)
13. [Component 9: Rate Limiting & Frequency Capping](#13-component-9-rate-limiting--frequency-capping)
14. [Component 10: Retry, DLQ & Delivery Tracking](#14-component-10-retry-dlq--delivery-tracking)
15. [Component 11: Scheduling & Digest Engine](#15-component-11-scheduling--digest-engine)
16. [Fan-out Strategies: Targeted vs Broadcast](#16-fan-out-strategies-targeted-vs-broadcast)
17. [CAP Theorem Positioning](#17-cap-theorem-positioning)
18. [Failure Modes & Mitigations](#18-failure-modes--mitigations)
19. [Scalability & Sharding](#19-scalability--sharding)
20. [Senior vs Staff Answer Differentiators](#20-senior-vs-staff-answer-differentiators)
21. [Interview Time Allocation](#21-interview-time-allocation)
22. [Quick-Reference Cheatsheet](#22-quick-reference-cheatsheet)

---

## 1. Problem Statement & Scope

A **notification system** is a shared internal platform that lets many producer services (order management, chat, fraud detection, marketing) send messages to end users across multiple channels — push, email, SMS, and in-app — without each producer having to integrate directly with APNs, FCM, Twilio, or SendGrid.

### What the interviewer is really testing

- Can you design a **multi-tenant, multi-channel fan-out system** that stays reliable under bursty, unpredictable load (a single event can target one user or fifty million)?
- Do you treat **idempotency, user preference, and rate limiting** as first-class concerns rather than afterthoughts?
- Can you reason about **third-party provider failure** as a normal operating condition, not an edge case?
- Do you make **explicit CAP theorem decisions per component** — and do you recognize that for this system, the "right" CAP answer sometimes depends on notification *category* (transactional vs. marketing), not just the component?

---

## 2. Requirements

### Functional Requirements

| Requirement | Notes |
|---|---|
| Accept notification requests from internal producer services | REST/gRPC API or async event consumption |
| Support multiple channels | Push (iOS/Android), Email, SMS, In-app/Web |
| Respect user preferences | Per-category opt-in/out, quiet hours, language |
| Template rendering & personalization | Variable substitution, localization |
| Deliver reliably, at-least-once, without duplicates | Idempotency guarantees |
| Track delivery status | Sent, delivered, opened, clicked, bounced, failed |
| Support scheduled and batched (digest) sends | Not just immediate fire |

### Non-Functional Requirements

| Requirement | Target |
|---|---|
| Scale | ~500M DAU, ~2.5B notifications/day across all channels |
| Latency (transactional) | P99 < 5s from API call to provider handoff |
| Latency (marketing/broadcast) | Minutes acceptable; throughput matters more than latency |
| Reliability | No producer-caused duplicate sends; no silent drops |
| Availability | 99.95%+ for the gateway; channel-level degradation isolated |
| Compliance | CAN-SPAM, TCPA, GDPR — opt-out must be honored reliably |

---

## 3. Capacity Estimation

```
Target: 500M DAU, 5 notifications/user/day average (all channels)

Daily volume:
  500,000,000 × 5 = 2.5 billion notifications/day

Average throughput:
  2.5B / 86,400 ≈ 28,935 notifications/second

Peak throughput (breaking-news / flash-sale broadcast, 8× average):
  ≈ 230,000 notifications/second at peak

Channel split (typical):
  Push:    60%  → ~17,400/s avg,  ~139,000/s peak
  Email:   25%  → ~7,200/s avg,   ~58,000/s peak
  SMS:      5%  → ~1,450/s avg,   ~11,600/s peak (cost-sensitive, throttled deliberately)
  In-app:  10%  → ~2,900/s avg,   ~23,000/s peak

Notification metadata storage (500 bytes/record):
  2.5B × 500B ≈ 1.25 TB/day  →  tiered storage required

Device token store:
  500M users × ~2 devices avg = 1B tokens × ~200B ≈ 200 GB

Dedup/idempotency store (Redis, 1-hour TTL window):
  ~230,000/s × 3600s × 100B/key ≈ 83 GB in-flight at peak → fits a Redis cluster

Kafka throughput:
  230,000 msgs/s × ~1KB avg ≈ 230 MB/s  →  ~24 partitions at 10MB/s each per topic
```

**Key insight:** Unlike the web crawler, storage is not the dominant cost here — **burst amplification** is. A single API call from a producer ("championship game just ended") can fan out to tens of millions of individual sends. The system must be designed so that one logical event never becomes one synchronous unit of work.

---

## 4. High-Level Architecture

```
                    ┌───────────────────────────────┐
                    │      Internal Producers        │
                    │ (Orders, Chat, Fraud, Marketing)│
                    └───────────────┬───────────────┘
                                    │ REST/gRPC/Kafka event + idempotency key
                                    ▼
                    ┌───────────────────────────────┐
                    │   Notification Gateway (API)   │
                    │  Auth · Schema validation ·    │
                    │  Request-level idempotency      │
                    └───────────────┬───────────────┘
                                    ▼
                    ┌───────────────────────────────┐
                    │ Preference & Subscription Svc  │
                    │ (opt-in/out, DND, quiet hours, │
                    │  frequency caps, locale)        │
                    └───────────────┬───────────────┘
                                    │ (filtered, allowed sends only)
                    ┌───────────────▼───────────────┐
                    │ Template & Personalization Svc │
                    │ (render content, i18n, vars)   │
                    └───────────────┬───────────────┘
                                    ▼
                    ┌───────────────────────────────┐
                    │   Router / Channel Selector    │
                    └──────┬─────────┬─────────┬─────┘
                           │         │         │
                 ┌─────────▼──┐ ┌────▼────┐ ┌──▼───────┐
                 │ Push Topic │ │Email Topic│ │SMS Topic │   (Kafka, partitioned by user_id)
                 └─────────┬──┘ └────┬────┘ └──┬───────┘
                           │         │         │
                 ┌─────────▼──┐ ┌────▼────┐ ┌──▼───────┐
                 │Push Workers│ │Email Wkrs│ │SMS Workers│
                 │→ APNs/FCM  │ │→SendGrid │ │→ Twilio  │
                 │ (+dedup    │ │  /SES    │ │ /Vonage  │
                 │  + retry)  │ │          │ │          │
                 └─────────┬──┘ └────┬────┘ └──┬───────┘
                           │         │         │
                           └─────────┴────┬────┘
                                          ▼
                              ┌───────────────────────┐
                              │  Delivery Tracking &   │
                              │  Retry / DLQ           │
                              │  (webhooks: delivered, │
                              │  opened, bounced)      │
                              └───────────┬───────────┘
                                          ▼
                              ┌───────────────────────┐
                              │  Analytics Store       │
                              │  (ClickHouse/BigQuery) │
                              └───────────────────────┘
```

**Kafka** again plays the same role it does across this guide series: durable decoupling between ingestion, routing, and delivery, with replay capability and back-pressure via consumer lag.

---

## 5. Component 1: Notification Gateway (API)

### What it is

The single entry point for every producer service. Two ingestion modes are typically supported:

1. **Synchronous REST/gRPC** — for low-volume, immediate sends (e.g., "your password was reset").
2. **Async event consumption** — the gateway itself subscribes to producer-owned Kafka topics (e.g., `order_shipped`) for high-volume, decoupled use cases.

### Responsibilities

```
1. Authenticate caller (mTLS or service JWT — internal service-to-service)
2. Validate payload against a registered schema per notification type
3. Require and check an idempotency key (producer-supplied or derived
   from event_id + event_type)
4. Persist the raw request durably (Kafka: raw-notification-requests)
   BEFORE any further processing — this is the write-ahead log
5. Apply a per-producer rate limit — a buggy producer must not be able
   to create a notification storm
```

### Staff-Level Gotcha

If the gateway processes requests synchronously end-to-end (render + preference check + provider call) before responding, a slow downstream provider stalls the API for every caller. **Decouple accept from process**: the gateway's job is to durably accept and hand off, not to guarantee delivery within the HTTP response. Return `202 Accepted` with a tracking ID; delivery status is queried separately or pushed via webhook back to the producer.

**Time to spend in interview:** ~3 minutes. State the write-ahead-log pattern and pivot to preferences.

---

## 6. Component 2: Preference & Subscription Service

### What it stores

| Field | Example |
|---|---|
| Channel opt-in/out per category | `marketing: push=off, email=on` |
| Do-not-disturb / quiet hours | `22:00–08:00 America/Los_Angeles` |
| Frequency cap | `max 3 marketing notifications/day` |
| Language/locale | `es-MX` |
| Global unsubscribe flag | Overrides all non-critical categories |

### The Category Distinction (Staff-level framing)

Not all notifications are equal under a preference check:

```
Transactional / security  → Cannot be opted out (password reset, fraud alert,
                              OTP). Legally and operationally must be delivered.
Operational                → Can typically be opted out per channel
                              (shipping updates, appointment reminders).
Marketing / engagement      → Fully subject to opt-out, frequency caps,
                              and quiet hours. Highest compliance risk.
```

A Senior-level design checks "is the user subscribed?" once. A Staff-level design threads the **category** through every downstream decision — CAP posture, retry aggressiveness, and quiet-hour enforcement all differ by category.

### Critical Timing Nuance

Preference must be checked **at send time**, immediately before dispatch to the provider — not only at enqueue time. For scheduled or digested notifications sent hours after the original request, a user may unsubscribe in that window. Re-checking only at enqueue creates a compliance gap.

```
Enqueue-time check:  fast filter, reduces wasted queue volume
Send-time check:     authoritative, must-not-skip, low-latency cache read
```

### Storage & Caching

- **Source of truth:** Postgres or DynamoDB, partitioned/sharded by `user_id`.
- **Read path:** Redis cache in front, TTL 60–300s, invalidated on preference write (cache-aside with active invalidation, not just TTL expiry, because opt-out correctness matters more than cache hit rate).

---

## 7. Component 3: Template & Personalization Service

### Rendering Pipeline

```
1. Fetch template by (notification_type, channel, locale)
2. Substitute variables (user name, order ID, amount, etc.)
3. Enforce channel-specific constraints:
     SMS   → 160 char segments (multi-part billing beyond that)
     Push  → title/body length limits per OS
     Email → HTML template + plain-text fallback (deliverability)
4. Attach category + priority metadata for downstream routing
```

### Staff-Level Optimization: Render Caching for Broadcast

For a broadcast notification ("Game 7 final score!") sent to 20M identical subscribers, re-rendering identical content 20M times is wasted CPU. Cache the rendered payload keyed by `(template_id, locale, variable_hash)` — most recipients on a broadcast share the same locale and variables, so the cache hit rate approaches 100% and per-recipient cost drops to a lookup instead of a render.

### Versioning & A/B Testing

Templates are versioned; the notification record stores which version was sent, enabling A/B experiment attribution and safe rollback of a bad template without reprocessing history.

---

## 8. Component 4: Router & Channel Selector

Given a filtered, rendered notification, decide which channel(s) to use and in what order.

### Routing Policy

```
Policy inputs:
  • User's channel preference ranking (e.g., push > email > SMS)
  • Notification category priority (security bypasses ranking → all channels)
  • Channel health (circuit breaker state per provider)
  • Cost (SMS is the most expensive per-message channel — used last/least)

Fallback chain example (transactional OTP):
  Try push (if device token exists and provider healthy)
    → wait for delivery confirmation window (e.g., 30s)
    → if undelivered, fall back to SMS
    → if SMS fails, fall back to email
```

### Why Fallback Chains Matter at Staff Level

A Senior-level answer picks one channel per notification. A Staff-level answer recognizes that for **critical** categories, single-channel delivery is a single point of failure — the fallback chain is itself a reliability mechanism, not just a UX nicety, and it must be time-bounded (don't wait forever for a push delivery receipt before trying SMS).

---

## 9. Component 5: Message Queue / Event Bus

### Partitioning Strategy

```
Partition key: user_id
  → preserves per-user ordering (e.g., "shipped" must never be
    processed after "delivered" for the same order)

Priority handled via separate topics, not partition priority:
  notifications-critical   (security, OTP, fraud)
  notifications-standard   (order updates, chat)
  notifications-bulk       (marketing, digests)

Weighted consumption across topics (same pattern as the crawler's
front-queue weighted round-robin): critical consumed with dedicated,
never-starved worker capacity; bulk absorbs backpressure first.
```

### Staff-Level Gotcha: The Hot-Key Problem

A single high-volume system account (e.g., an official sports league account with 50M followers) triggers a single event that must fan out to 50M user-partitioned messages. If fan-out happens synchronously inside the producer's request path, one partition's worth of routing logic becomes a bottleneck. This is solved structurally — see [Section 16, Fan-out Strategies](#16-fan-out-strategies-targeted-vs-broadcast) — not by just adding more partitions.

---

## 10. Component 6: Channel Workers & Provider Integration

### Push (APNs / FCM)

```
APNs:  HTTP/2, token-based (JWT) or certificate auth, per-device token
FCM:   HTTP, service-account auth, supports multicast (up to 500
       tokens per batch call) — batch aggressively to cut request volume

Both return per-token status codes — a single batch call can partially
succeed. Parse per-token results; don't treat the batch as atomic.
```

### Email (SendGrid / SES / Mailgun)

```
Provider API call → async webhook callbacks for:
  delivered, bounced (hard/soft), opened, clicked, spam-complaint

Hard bounce or spam complaint → immediately suppress future sends to
that address (compliance-critical, not just best-effort).
```

### SMS (Twilio / Vonage)

```
Highest per-message cost → lowest default priority; used for
critical/fallback paths, not routine engagement.
Carrier-specific rate limits and country-code routing rules apply —
these are more restrictive and more variable than push/email limits.
```

### In-App / Web Push

```
Delivered via WebSocket/SSE from a connection-gateway service if the
user is currently online (presence-aware).
If offline: persist as an unread in-app notification, delivered on
next app open — this is the "fan-out on read" analog for a single user,
distinct from the broadcast fan-out-on-read pattern in Section 16.
```

### Multi-Provider Redundancy (Staff-level)

For channels where delivery is business-critical, integrate two providers behind a **circuit breaker** (see the Circuit Breaker guide for the state-machine detail): if the primary provider's error rate crosses a threshold, open the circuit and route new traffic to the secondary provider automatically, rather than queuing behind a degraded dependency.

---

## 11. Component 7: Device Token Management

### Lifecycle

```
Register:    client sends token on install / token refresh
Store:       {user_id, device_id, token, platform, app_version, last_seen}
Invalidate:  APNs/FCM return "Unregistered" / "InvalidRegistration" in
             the send response (or via a separate feedback service) →
             prune immediately

Multi-device: a user may have N tokens. Sending N duplicate pushes for
one logical notification is often undesirable — track a shared
notification_id and, optionally, cross-device read receipts so that
reading on one device can cancel a pending push to others.
```

### Why Prompt Pruning Matters

Continuing to send to invalid tokens doesn't just waste throughput — high invalid-token rates can affect a sending app's standing with the provider (deliverability reputation), the same way a poorly-behaved crawler gets IP-blocked. Token hygiene is a first-class operational concern, not cleanup work.

---

## 12. Component 8: Idempotency & Deduplication

This is the direct analog of the web crawler's two-layer dedup service — but here, a miss is user-visible spam rather than wasted compute.

### Layer 1: Request-Level Idempotency (Gateway)

```
Producer supplies (or gateway derives) an idempotency key, e.g.:
  key = hash(event_id + event_type + notification_category)

Gateway checks a Redis SET (TTL-bounded, e.g. 24h) using SETNX:
  Key exists → reject/no-op, return the original tracking ID
  Key absent → accept, write key, proceed

Purpose: absorb producer-side retries (network timeout → producer
retries the same logical event) without creating duplicate notifications.
```

### Layer 2: Delivery-Level Dedup (Worker)

```
Before calling the third-party provider, check a short-TTL "attempted"
marker keyed by notification_id, using SETNX:
  Guards against Kafka's at-least-once redelivery — a worker can crash
  after calling the provider but before committing its consumer offset,
  causing the same message to be reprocessed.

Ordering that matters:
  1. SETNX attempted-marker
  2. Call provider
  3. Commit Kafka offset
  (If step 2 fails, the marker's TTL expiry allows a clean retry;
   don't commit the offset until the provider call has a terminal result.)
```

### Dedup Decision Matrix

| Request-level key seen? | Delivery marker seen? | Action |
|---|---|---|
| Yes | — | Reject at gateway; return existing tracking ID |
| No | Yes | Skip provider call; treat as already-attempted |
| No | No | Proceed: render, route, send, mark |

---

## 13. Component 9: Rate Limiting & Frequency Capping

Three independent layers, each protecting a different resource:

```
1. Per-producer rate limit (Gateway)
   Protects the platform from a misbehaving internal service.

2. Per-user frequency cap (Preference/Frequency service)
   Sliding-window counter in Redis, e.g. "max 3 marketing
   notifications per user per day." Protects the user from fatigue
   and protects the business from opt-outs/uninstalls.

3. Per-provider rate limit (Channel worker)
   Token-bucket per provider account, respecting Twilio/APNs/FCM
   quotas — conceptually identical to the web crawler's per-domain
   politeness delay: the "domain" here is the provider account.
```

### Global Circuit-Breaker Throttle

During an incident (e.g., a bug generating a runaway loop of order-update events), an on-call engineer needs a global kill switch that can pause a category or producer instantly — this is an operational requirement, not just a design nicety, and should be called out proactively.

---

## 14. Component 10: Retry, DLQ & Delivery Tracking

### Retry Policy

| Failure type | Example | Action |
|---|---|---|
| Transient | 5xx, timeout, provider rate-limited | Exponential backoff + jitter, retry up to N times |
| Permanent | Invalid token, hard bounce, invalid number | No retry; mark dead; prune source record |
| Ambiguous | Provider returns no clear status | Retry with backoff; escalate to DLQ after N attempts |

### Dead Letter Queue

Messages exceeding max retries land in a DLQ topic, monitored by automated alerting (a sustained rise in DLQ volume for one provider is often the earliest signal of a provider outage — earlier than the provider's own status page).

### Delivery Tracking Pipeline

```
Provider webhooks (delivered, opened, clicked, bounced, complained)
    → Delivery Events topic (Kafka)
    → Aggregation into analytics store (ClickHouse/BigQuery)
    → Feedback loops:
         bounce/complaint → auto-suppress future sends (compliance)
         low open-rate on a template → surfaced to marketing team
```

**Staff-level note:** treat webhook ingestion lag itself as a monitored SLO. If bounce/complaint events are delayed, the auto-suppression feedback loop is delayed too — a silent compliance risk, not just a metrics-freshness issue.

---

## 15. Component 11: Scheduling & Digest Engine

### Scheduled Sends

```
"Send at 9am user-local time" requires timezone-aware scheduling:
  Store target_send_time in UTC, computed from the user's stored
  timezone at request time (not at send time — timezone changes,
  e.g. travel, shouldn't retroactively reschedule an already-queued send).

Scheduling mechanism options:
  • Min-heap scheduler service keyed on target_send_time (same pattern
    as the crawler's domain heap keyed on next_ok_ts)
  • Kafka delay-queue pattern (tiered topics with consumer delay)
  • Managed delay queue (e.g., SQS delay, up to its native limits)
```

### Digest / Batching Engine

```
Purpose: prevent notification fatigue from high-frequency, low-value
events ("5 people liked your post" instead of 5 separate pushes).

Window: accumulate qualifying events per user over a configurable
window (e.g., 30 min), then emit one aggregated notification.

Eligibility rule (critical): only engagement/social categories are
digest-eligible. Transactional and security notifications must NEVER
be batched or delayed — this eligibility check belongs in the
Preference/Category layer, not bolted onto the digest engine itself.
```

---

## 16. Fan-out Strategies: Targeted vs Broadcast

Directly analogous to the fan-out-on-write vs. fan-out-on-read tradeoff from the Twitter guide, applied to notifications.

### Fan-out on Write (default, targeted)

```
Event occurs → immediately enqueue one message per recipient.
Good for: normal-scale events (1 order → 1 recipient; 1 friend
request → 1 recipient). Low latency, simple.
```

### Fan-out on Read (broadcast / celebrity case)

```
Problem: a single API call ("breaking news") targeting 50M
recipients must not synchronously generate 50M queue messages in
the producer's request path — this recreates the exact hot-partition
and thundering-herd problem the crawler guide addresses via
hot-domain sub-queue splitting.

Solution:
  1. Producer publishes ONE broadcast event (recipient list is
     implicit — "all subscribers of topic X").
  2. A horizontally-scaled Fan-out Worker tier pulls the subscriber
     list in paginated batches from the subscription store.
  3. Each batch is pushed to per-channel queues at a rate-limited
     pace, bounded by downstream provider capacity — not by how fast
     the subscriber list can be read.

Result: the API call returns immediately; the fan-out itself is a
background, rate-controlled, horizontally-scalable job — not a
request-time liability.
```

**Staff-level signal:** explicitly naming this as the same class of problem as celebrity fan-out in a social feed system, and proposing the batch-and-throttle worker tier rather than "just add more Kafka partitions."

---

## 17. CAP Theorem Positioning

The key nuance for this system: CAP posture sometimes depends on **notification category**, not only on the component. State both dimensions explicitly.

| Component | CAP Choice | Reasoning |
|---|---|---|
| Notification Gateway ingestion log (Kafka) | **AP** | Write-ahead log; availability of ingestion matters more than immediate consistency. |
| Preference/Subscription store — **marketing category** | **CP** | Under partition, fail closed (don't send). A missed opt-out is a compliance and trust violation; a delayed marketing send is not. |
| Preference/Subscription store — **transactional category** | **AP-leaning** | Under partition, default-allow may be acceptable per business policy — a missed password-reset email is worse than a rare, brief preference-staleness window. This is a business decision to state explicitly, not assume. |
| Idempotency/dedup store (Redis) | **CP** | Same logic as the crawler's Bloom filter: a false negative here means a duplicate, user-visible notification. Prefer to reject/pause over risking a duplicate. |
| Device token store | **AP** | A stale token just causes one failed send, corrected on next feedback-loop prune. Low cost of staleness. |
| Template store | **AP** | Aggressively cached; a briefly stale template is a non-issue. |
| Delivery tracking / analytics store | **AP** | Eventual consistency is fine for dashboards and engagement metrics. |
| Rate limiter counters (Redis) | **AP** (general) / **CP** (legally-mandated caps, e.g. TCPA SMS consent limits) | Ordinary frequency caps tolerate slight overage; legally mandated caps should fail closed. |
| Scheduler coordination (leader election, ZooKeeper/etcd) | **CP** | Duplicate schedulers firing the same scheduled batch twice is a correctness bug, not a tolerable inconsistency. |

---

## 18. Failure Modes & Mitigations

| Failure | Symptom | Detection | Mitigation |
|---|---|---|---|
| Third-party provider outage (APNs/FCM down) | Push error rate spikes | Provider error-rate monitoring | Circuit breaker opens; queue backlog retained in Kafka; route critical categories to fallback channel |
| Broadcast fan-out storm | Millions of messages generated synchronously | Sudden partition/queue depth spike | Fan-out-on-read with paginated, rate-limited worker tier (Section 16) |
| Duplicate notification sent | User complaint, spam flag, provider reputation hit | Dedup-marker miss rate, user reports | Two-layer idempotency (request + delivery level) |
| Stale/invalid device token | Wasted sends; provider reputation penalty | Invalid-token response rate per app | Prune on feedback-loop signal; don't retry permanent failures |
| Preference race (send after opt-out) | Compliance/legal risk | Complaint rate, audit log diff | Send-time (not just enqueue-time) preference recheck; short-TTL cache with active invalidation |
| Notification stuck in infinite retry | Resource drain, delayed queue | Retry-count distribution | Max retry cap + exponential backoff + DLQ |
| Wrong-timezone delivery | User complaint, uninstall risk | Off-hours delivery rate per locale | Timezone computed at request time; stored per user, not assumed from IP |
| Kafka hot-key from a mega-account | One partition/consumer overloaded | Per-partition lag skew | Route mega-account broadcasts through the fan-out worker tier, not direct per-user partitioning |
| Template render failure (missing variable) | Blank/broken content delivered | Schema validation failures | Validate payload against template schema pre-enqueue; safe default fallback content |
| Delivery webhook lag | Suppression/analytics feedback delayed | Webhook consumer lag | Treat as a monitored SLO; alert on lag, not just on error rate |
| Multi-region latency for global users | Slow delivery for distant users | Regional latency percentiles | Regional deployment with local provider endpoints; preference store read replicas |
| PII exposure in logs/tracking store | Compliance/security incident | Access audit, log scanning | Redact/encrypt PII at rest and in transit; strict access control on delivery tracking data |

---

## 19. Scalability & Sharding

### Sharding Strategy

```
Preference/Subscription store: shard by hash(user_id)
Kafka topics:                  partition by user_id (per-user ordering)
                                — except mega-account broadcasts, routed
                                  through the dedicated fan-out tier
Channel workers:                scale independently per channel — push
                                 volume is typically 6–10× SMS volume;
                                 provisioning them identically wastes cost
```

### Horizontal Scaling Plan

| Component | Scaling Strategy |
|---|---|
| Notification Gateway | Stateless; scale behind a load balancer |
| Preference cache (Redis) | Cluster mode, sharded by user_id |
| Kafka | Add partitions per topic; scale consumer groups in lockstep |
| Channel workers | Independent auto-scaling group per channel |
| Fan-out worker tier | Scales with broadcast frequency, not baseline traffic — separate capacity plan |
| Delivery tracking store | Add nodes; time-series partitioning by ingestion date |

### Multi-Region

Regional notification services with local provider endpoints reduce cross-region latency for a global user base. Preference data is replicated with regional read replicas; writes route to the user's home region to keep opt-out changes strongly consistent at the source of truth.

---

## 20. Senior vs Staff Answer Differentiators

### Senior-level expectations

- Correctly identify the major components (gateway, preferences, templates, queue, channel workers)
- Explain basic retry logic and mention a DLQ
- Know that push/email/SMS use different third-party providers
- Mention rate limiting in general terms
- Provide basic capacity math

### Staff-level expectations (additional)

| Topic | What Staff-level adds |
|---|---|
| Idempotency | Two independent layers (request-level and delivery-level), and why each exists separately |
| Preference service | CAP posture that differs **by notification category**, not a single blanket answer |
| Fan-out | Explicit fan-out-on-write vs. fan-out-on-read distinction; broadcast handled by a dedicated rate-limited worker tier |
| Provider integration | Multi-provider redundancy behind a circuit breaker for critical channels |
| Rate limiting | Three independent layers (producer, user, provider), each protecting a different failure mode |
| Scheduling/digests | Explicit digest-eligibility rule tied to category, never applied to transactional sends |
| Device tokens | Feedback-loop pruning as an operational necessity, not cleanup |
| Delivery tracking | Webhook lag treated as its own monitored SLO, tied to compliance suppression correctness |
| CAP theorem | Explicit AP vs CP decision per component **and per category** where they diverge |
| Failure modes | Proactively enumerate broadcast storms, hot-key partitions, and preference races |

### The single most important Staff differentiator

**Recognizing that "notification system" is really several different reliability problems wearing one API** — a transactional OTP, a marketing blast, and a broadcast to fifty million subscribers have almost nothing in common operationally, even though they enter through the same gateway. A Staff-level design threads that distinction through preferences, CAP posture, retry aggressiveness, and fan-out strategy consistently, rather than designing one pipeline and noting exceptions when asked.

---

## 21. Interview Time Allocation

For a 45-minute system design session:

| Phase | Time | Focus |
|---|---|---|
| Requirements & scope | 5 min | Confirm channels, categories (transactional vs marketing), scale target |
| Capacity estimation | 5 min | Throughput avg/peak, channel split, storage |
| High-level architecture | 5 min | Draw all components; name the message bus (Kafka) |
| Preferences & idempotency (deep dive) | 10 min | Category-based CAP posture; two-layer dedup |
| Fan-out strategies | 8 min | Fan-out-on-write vs on-read; broadcast worker tier |
| Channel workers & provider integration | 7 min | Multi-provider redundancy, circuit breaker, token management |
| Failure modes | 5 min | Provider outage, broadcast storm, preference race |

**What to cut if short on time:** Digest engine detail and multi-region depth. **Never cut:** Idempotency/dedup, category-based preference handling, and fan-out strategy.

---

## 22. Quick-Reference Cheatsheet

```
KEY PATTERNS
────────────
Idempotency:      Two-layer — request-level (gateway, SETNX) + delivery-level (worker, SETNX)
Fan-out:          Fan-out-on-write (targeted) vs fan-out-on-read (broadcast, rate-limited worker tier)
Rate limiting:    Three independent layers — per-producer, per-user, per-provider (token bucket)
Preference check: Enqueue-time (fast filter) AND send-time (authoritative, must-not-skip)
Digest eligibility: Category-gated — transactional/security NEVER batched

KEY NUMBERS
───────────
500M DAU, 5 notifications/user/day  →  2.5B/day
Average throughput  ≈ 28,935/s   |   Peak (8×)  ≈ 230,000/s
Channel split: Push 60% · Email 25% · SMS 5% · In-app 10%
Notification metadata: ~1.25 TB/day
Device token store: ~200 GB (1B tokens)

KEY SYSTEMS
───────────
Message bus:      Kafka (partitioned by user_id; broadcast bypasses direct partitioning)
Preference store: Postgres/DynamoDB + Redis cache (short TTL, actively invalidated)
Dedup store:      Redis (SETNX, TTL-bounded)
Push:             APNs (iOS) / FCM (Android, cross-platform, batch up to 500/call)
Email:            SendGrid / SES / Mailgun (webhook-driven delivery tracking)
SMS:              Twilio / Vonage (cost-sensitive, lowest default priority)
Analytics:        ClickHouse / BigQuery
Coordination:     ZooKeeper/etcd (CP, scheduler leader election)

CAP DECISIONS
─────────────
AP:  Ingestion log, device tokens, template store, analytics store, ordinary rate-limit counters
CP:  Dedup store, marketing-category preferences, legally-mandated rate caps, scheduler coordination
Depends on category: transactional preferences (business policy call, state explicitly)

STAFF-LEVEL SIGNALS TO HIT
───────────────────────────
✓ Two-layer idempotency (request + delivery level)
✓ CAP posture stated per component AND per notification category
✓ Fan-out-on-read broadcast tier for mega-accounts (not naive per-user fan-out)
✓ Multi-provider redundancy behind a circuit breaker for critical channels
✓ Three independent rate-limiting layers with distinct purposes
✓ Send-time preference recheck, not just enqueue-time
✓ Digest eligibility gated by category, never applied to transactional sends
✓ Device token feedback-loop pruning as an operational necessity
✓ Webhook ingestion lag monitored as its own SLO (compliance-linked)
✓ Proactive failure mode enumeration (don't wait to be asked)
```

---

*Guide compiled for Senior & Staff-level system design interview preparation.*
*Topics: Notification System, Fan-out, Idempotency, Rate Limiting, CAP Theorem, Multi-Channel Delivery.*
