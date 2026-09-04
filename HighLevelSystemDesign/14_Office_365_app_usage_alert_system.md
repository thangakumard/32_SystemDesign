# Office 365 App Usage Alert System — System Design Interview Guide
### Senior & Staff Engineer Level

---

## Table of Contents

1. [Problem Statement & Scope](#1-problem-statement--scope)
2. [Clarifying Questions to Ask](#2-clarifying-questions-to-ask)
3. [Requirements](#3-requirements)
4. [Candidate Approaches Compared](#4-candidate-approaches-compared)
5. [Capacity Estimation](#5-capacity-estimation)
6. [High-Level Architecture](#6-high-level-architecture)
7. [Component 1: Local Telemetry Agent & Local-First Suppression](#7-component-1-local-telemetry-agent--local-first-suppression)
8. [Component 2: Usage Ingestion Pipeline](#8-component-2-usage-ingestion-pipeline)
9. [Component 3: last_used_at Store](#9-component-3-last_used_at-store)
10. [Component 4: Alert Computation — Lazy vs Active](#10-component-4-alert-computation--lazy-vs-active)
11. [Component 5: Delay-Queue Scheduler (Push Variant)](#11-component-5-delay-queue-scheduler-push-variant)
12. [Component 6: Policy/Config Service](#12-component-6-policyconfig-service)
13. [Component 7: Caching Layer](#13-component-7-caching-layer)
14. [Cross-Device Reconciliation](#14-cross-device-reconciliation)
15. [CAP / PACELC Theorem Positioning](#15-cap--pacelc-theorem-positioning)
16. [Failure Modes & Mitigations](#16-failure-modes--mitigations)
17. [Privacy, Compliance & Fail-Safe Defaults](#17-privacy-compliance--fail-safe-defaults)
18. [Scalability & Sharding](#18-scalability--sharding)
19. [Senior vs Staff Answer Differentiators](#19-senior-vs-staff-answer-differentiators)
20. [Interview Time Allocation](#20-interview-time-allocation)
21. [Quick-Reference Cheatsheet](#21-quick-reference-cheatsheet)

---

## 1. Problem Statement & Scope

Design a system that tracks per-user, per-app usage across the Office 365 suite (Word, Excel, PowerPoint, Outlook, Teams, OneNote, etc.) and surfaces a **taskbar alert** when a given app hasn't been used for **X days**. If the user opens that app, the alert must **not** appear (or must clear immediately if already showing).

### What the interviewer is really testing

- Can you recognize that this is fundamentally a **state-sync problem**, not an **event-notification problem**?
- Do you know when to **compute lazily** vs. **precompute proactively**, and why that choice matters at hundreds-of-millions-of-users scale?
- Can you design a **local-first / eventually-consistent** client-server split where correctness requirements differ sharply between "suppress instantly" and "detect neglect"?
- Do you avoid over-engineering consistency into a system where the cost of being wrong is genuinely low?

### The single most important Staff-level differentiator

> **Alert state is a derived value, not a stored fact.** `alert = (now − last_used_at) > threshold` can be computed at read time from a single timestamp. This eliminates the need for any daily sweep across the entire user base for the baseline "badge" requirement — you only pay computation cost when someone actually looks at their taskbar. A Senior-level answer reaches for "a cron job scans everyone and flags stale apps." A Staff-level answer recognizes that's billions of wasted row-scans a day for a UI element most users never even glance at, and reaches for on-demand derivation instead — reserving active scheduling only for the harder sub-problem of *proactive push notifications* (Section 11).

---

## 2. Clarifying Questions to Ask

The prompt as given is intentionally underspecified — a Staff candidate surfaces this instead of silently assuming:

| Question | Why it matters |
|---|---|
| Is this a **persistent badge** (like the OneDrive taskbar icon) or a **one-time push notification/toast**? | Badge = pure lazy read. Toast = needs active threshold-crossing detection. Very different architectures. |
| Is the threshold **global**, per-app, per-tenant (admin policy), or per-user configurable? | Determines whether Policy Service needs per-tenant override storage. |
| Does usage on **any device** count, or is this per-device? | Determines whether cross-device aggregation is required (assume yes — it's the realistic case). |
| What counts as "used" — app **opened**, or opened **and actively focused** for some duration? | Prevents gaming/false-positives (e.g., accidental double-click). |
| Is a **brand-new install** exempt on day 1? | Avoids nagging users who haven't had a chance to use the app yet. |
| Are there **privacy/telemetry opt-out** requirements (enterprise compliance, GDPR)? | Drives fail-safe design (Section 17). |

For the rest of this guide, I'll assume: cross-device aggregation, per-app configurable threshold with tenant-level admin override, "opened" counts as usage (with a note on a focus-duration refinement), and a badge as the primary surface with a push variant as an extension.

---

## 3. Requirements

### Functional Requirements

| Requirement | Notes |
|---|---|
| Track last-used timestamp per (user, app), aggregated across all the user's devices | Core state |
| Show taskbar alert if unused for ≥ X days | X configurable per app/tenant |
| Suppress/clear alert immediately if used "just now" | Must feel instant on the device where it happened |
| Support per-app, per-tenant threshold configuration | Admin policy |
| New installs should not immediately trigger a false "unused" alert | Install-date anchor, not epoch |
| (Extension) Proactive push notification the moment a threshold is crossed | Optional; changes architecture (Section 11) |

### Non-Functional Requirements

| Requirement | Target |
|---|---|
| Scale | ~300–400M O365 seats × ~20–30 tracked apps |
| Local suppression latency | Effectively instant (< 1s, no server round-trip required) |
| Cross-device sync latency | Minutes is acceptable — this is a nudge, not a safety system |
| Ingestion durability | At-least-once; duplicates must be harmless |
| Read cost | Must not scale with total user population — only with actual taskbar checks |
| Privacy | Must degrade fail-safe (never alert) when telemetry is unavailable/opted out |

---

## 4. Candidate Approaches Compared

This is worth walking through explicitly in an interview — it shows you've considered the design space, not just landed on an answer.

### Approach A: Scheduled Batch Sweep (naive, Senior-level default)

A nightly cron job scans every (user, app) pair, computes staleness, and writes an `alert=true/false` flag that the client simply reads.

- **Pros:** Simple mental model. Client read is trivial. Works fine at small scale.
- **Cons:** At O365 scale, you're sweeping **~10 billion rows/day** (400M users × 25 apps) to update a UI badge that most users never look at on most days. Also introduces up to 24h of staleness on the "just used → suppress" path unless paired with a local override anyway — so you end up needing the local override *and* paying for the sweep.
- **Verdict:** Rejected as the primary mechanism. Only worth reconsidering if you also need proactive push (see Approach C) — and even then, a targeted mechanism beats a full sweep.

### Approach B: Lazy / On-Read Computation (recommended baseline)

Store only `last_used_at`. Compute `alert = (now − last_used_at) > threshold(app, tenant)` at the moment the client asks.

- **Pros:** Zero wasted computation — cost scales with actual taskbar checks, not total population. Always reflects the current `last_used_at` with no separate staleness window of its own. No scheduler, no coordinator, minimal moving parts.
- **Cons:** Cannot originate an unsolicited push — nobody is "reading" at the exact moment a threshold crosses, so this alone can't drive a toast notification.
- **Verdict:** Best fit for the stated problem (a persistent taskbar badge). This is the recommended baseline design, detailed in Sections 6–10.

### Approach C: Delay-Queue / Min-Heap Scheduled Wake-Up (for proactive push)

Mirrors a pattern you've likely already designed elsewhere: a **min-heap keyed on next-eligible-time**, exactly like the Web Crawler's Domain Heap keyed on `next_ok_ts`. Insert `(user, app)` with `next_eligible = last_used_at + threshold`; reinsert with a fresh time on every new usage event; a scheduler pops only entries that are actually due.

- **Pros:** Enables proactive notifications without sweeping everyone — you wake up exactly once per genuine threshold crossing, and cost scales with event rate, not population size.
- **Cons:** More infrastructure (a real delay-queue/timer-wheel primitive, sharded to avoid a single coordinator bottleneck); needs idempotent dedup for duplicate/late fires.
- **Verdict:** Layer this on top of Approach B **only if** product requires unsolicited toasts, not just a badge. Detailed in Section 11.

### Approach D: Precompute + Push-as-Invalidation-Hint (hybrid, best for production)

Keep Approach B as the source of truth, but use a lightweight pub/sub signal — realistically, O365 already runs a notification hub for Teams/Outlook activity feeds — to tell an actively-connected client "your cached alert state may have changed, refetch," instead of polling on a fixed interval.

- **Pros:** Cuts polling volume for long-lived client sessions while keeping the underlying computation cheap and always-correct if the hub is unavailable.
- **Cons:** Adds a dependency on existing notification infrastructure; still needs a polling fallback for resilience, so it's additive complexity, not a replacement.
- **Verdict:** The strongest real-world answer — reuse existing infra as an optimization on top of the lazy-eval foundation, not as a replacement for it.

**Recommendation to state in an interview:** lead with Approach B as the correct default, mention Approach A only to explain why you're rejecting it, and bring up C/D proactively when discussing extensions — this is exactly the kind of "I considered alternatives and can defend the choice" signal that separates Staff from Senior.

---

## 5. Capacity Estimation

```
Assumptions (illustrative, not pinned facts):
  Users:            ~400M O365 seats
  Tracked apps:     ~25 per user (Word, Excel, PPT, Outlook, Teams, OneNote, ...)
  Total (user,app) state rows:  400M × 25 = 10B rows

Row size (last_used_at record):
  user_id (16B) + app_id (4B) + timestamp (8B) + install_date (8B) ≈ 40-60B/row
  Total store size: 10B rows × ~60B ≈ 600 GB  →  comfortably fits a KV store

Usage events (write path):
  Avg 20 app-opens/user/day across all apps → 8B events/day
  8,000,000,000 / 86,400 ≈ 92,600 events/sec average
  Peak (login-hour burst, ~8x average) ≈ 740,000 events/sec

Alert queries (read path):
  Assume active devices poll every ~30 min during an ~8hr active window
  400M users × 16 polls/day ≈ 6.4B queries/day ≈ 74,000 req/sec average
  Cache absorbs the majority (Section 13) — primary store sees a fraction of this
```

**Key insight:** Unlike the crawler's storage-dominated cost profile, this system is **read-and-write-request dominated, not storage dominated** — the state itself is tiny (sub-terabyte), but the request volume from hundreds of millions of devices is the real engineering problem, which is why caching and the lazy-eval decision matter more here than storage tiering does.

---

## 6. High-Level Architecture

```
                     ┌────────────────────────────┐
                     │  O365 App (Word/Excel/PPT…) │
                     └──────────────┬─────────────┘
                                    │ on open / focus
                                    ▼
                     ┌────────────────────────────┐
                     │  Local Telemetry Agent       │
                     │  (per-device background svc) │
                     │  • local_last_used[app]=now  │◄── instant local badge clear
                     │  • enqueue usage event        │
                     └──────────────┬─────────────┘
                                    │ async, batched, retried
                                    ▼
                     ┌────────────────────────────┐
                     │  Usage Ingestion API          │
                     └──────────────┬─────────────┘
                                    ▼
                     ┌────────────────────────────┐
                     │  Durable Event Log (Kafka)    │
                     │  partitioned by user_id        │
                     └──────────────┬─────────────┘
                                    ▼
                     ┌────────────────────────────┐
                     │  Stream Aggregator             │
                     │  last_used_at = max(old, new)  │
                     └──────────────┬─────────────┘
                                    ▼
                     ┌────────────────────────────┐
                     │  last_used_at Store (KV)       │
                     │  partition key: user_id        │
                     │  sort key: app_id               │
                     └──────────────┬─────────────┘
                                    │
                    ┌───────────────┴────────────────┐
                    ▼                                  ▼
        ┌────────────────────┐          ┌─────────────────────────┐
        │  Per-user Cache      │          │  Policy/Config Service   │
        │  (Redis, short TTL)  │          │  thresholds per app/     │
        └──────────┬──────────┘          │  tenant/user override    │
                    │                     └────────────┬────────────┘
                    └───────────────┬───────────────────┘
                                    ▼
                     ┌────────────────────────────┐
                     │  Alert Query API              │
                     │  alert = (now − last_used)    │
                     │          > threshold          │
                     └──────────────┬─────────────┘
                                    │ poll (or push-hint, Approach D)
                                    ▼
                     ┌────────────────────────────┐
                     │  Local Telemetry Agent         │
                     │  renders taskbar badge          │
                     └────────────────────────────┘
```

Two distinct paths, with very different consistency requirements:

- **Write path (usage events):** high volume, append-only, idempotent — optimized for durability and throughput.
- **Read path (alert queries):** high volume, latency-sensitive, tolerant of staleness — optimized for cheap, cacheable, derived reads.

---

## 7. Component 1: Local Telemetry Agent & Local-First Suppression

### Why local-first matters

The requirement "if the user used the app just now, alert should not be shown" is the **hardest latency constraint in the system** — and the cheapest to satisfy, if you solve it locally instead of round-tripping to the server.

```
On app launch (local device):
  1. Local Telemetry Agent intercepts/observes the launch event
  2. Immediately sets local_last_used[app] = now
  3. Immediately clears any badge currently shown for that app on this device
  4. Asynchronously enqueues a usage event for upload (batched, retried, offline-tolerant)
```

The badge clears with **zero server dependency** — correctness for "just used → suppress" never depends on network latency, ingestion pipeline health, or store consistency. This is the same design instinct as the Web Crawler's per-fetcher local DNS cache: put the latency-critical decision as close to the point of use as possible, and let the durable/global system catch up asynchronously.

### What gets uploaded

A minimal event: `{user_id, app_id, device_id, client_timestamp, server_will_stamp_on_ingest}`. The **server-assigned ingestion timestamp**, not the client timestamp, is the source of truth for cross-device merge — this avoids clock-skew bugs (Section 16).

### Minimum-use threshold (refinement)

To avoid an accidental double-click counting as "real use," the agent can require the app to hold foreground focus for some minimum duration (e.g., 15–30s) before emitting the usage event. This is a product-tunable knob, not a correctness requirement — get the simple version working first, mention this as a refinement.

---

## 8. Component 2: Usage Ingestion Pipeline

### Design

```
Usage Ingestion API (stateless, authenticated via AAD/Entra ID token)
    → validates event, attaches server timestamp
    → writes to Kafka, partitioned by user_id
    → Stream Aggregator consumes, applies: last_used_at = max(existing, new)
    → writes to KV store
```

### Why idempotent max-merge is the key correctness property

Unlike a payments system that needs exactly-once processing, usage events only ever need to move a timestamp **forward**. Duplicate delivery, out-of-order delivery, or replay are all harmless under a `max()` merge — the worst case is a redundant write, never an incorrect state. This lets the pipeline run at-least-once delivery (cheap, simple) instead of requiring exactly-once semantics (expensive, complex) — a deliberate, explicit tradeoff worth stating out loud in an interview.

### Partitioning

Partition Kafka by `user_id`, not `(user_id, app_id)` — this keeps all of one user's app events ordered relative to each other on one partition, which simplifies the aggregator (no cross-partition merge needed for a single user's state) and keeps partition count manageable (hundreds of millions of users still shard cleanly by hash).

---

## 9. Component 3: last_used_at Store

### Schema

```
Table: app_usage
  Partition key: user_id
  Sort key:      app_id
  Attributes:
    last_used_at     TIMESTAMP
    first_seen_at     TIMESTAMP   (install/first-launch date — see Section 16)
    device_last_used  STRING      (optional: which device, for diagnostics)

Example row:
  { user_id: "u123", app_id: "powerpoint", last_used_at: "2026-08-30T14:02:00Z",
    first_seen_at: "2024-01-10T09:00:00Z" }
```

### Store choice

A wide-row KV store — DynamoDB, Cosmos DB, or Cassandra — partitioned by `user_id`. This is the same collocation pattern as the Web Crawler's `url_metadata` table partitioned by `domain`: **all of one user's app-usage rows land on the same partition**, so "give me alert state for all of this user's installed apps" is a single-partition read, not a scatter-gather across shards.

At ~600GB total (Section 5), this easily fits standard KV infrastructure with room to spare — no tiering strategy needed, unlike the Crawler's 100TB/day HTML problem.

---

## 10. Component 4: Alert Computation — Lazy vs Active

This is the architectural crux of the whole design (Section 4, Approach B).

```
Alert Query API:
  Input:  user_id, list of installed app_ids, current time (server clock)
  For each app_id:
    row = KV_store.get(user_id, app_id)
    threshold = PolicyService.get_threshold(tenant_id, app_id)   # cacheable
    if row is null:
        alert = false   # never seen ≠ neglected — see first_seen_at handling below
    elif (now - row.first_seen_at) < threshold and row.last_used_at is null:
        alert = false   # new install grace period
    else:
        unused_for = now - row.last_used_at
        alert = unused_for > threshold
  Return: { app_id: {alert: bool, last_used_at, unused_days} }
```

No sweep, no scheduler, no precomputed flag to keep in sync. The alert is recomputed fresh every time it's asked for, from a single stored timestamp plus a policy lookup — both cheap, both cacheable.

**Why this beats precomputing for this specific problem:** precomputing only pays off when the number of *reads* of a value exceeds the number of *writes* to the underlying data by a wide margin, or when the read is expensive to compute. Here, the computation is a subtraction and a comparison — cheaper than the write path that would be needed to keep a precomputed flag in sync. Precomputing would be solving a problem this system doesn't have.

---

## 11. Component 5: Delay-Queue Scheduler (Push Variant)

Only needed if product wants an **unsolicited toast** ("You haven't opened Excel in 30 days") rather than a passive badge someone has to notice on their own.

### Design — reuses the Web Crawler's Domain Heap pattern

```
On each usage event for (user, app):
   next_eligible[user,app] = now + threshold(app)
   reinsert (user,app) into a per-shard min-heap keyed by next_eligible
   (this naturally "cancels" any earlier pending fire — same as the crawler
    reinserting a domain into its heap with a new next_ok_ts after each fetch)

Scheduler loop (per shard):
   1. Peek top of heap → (user,app) with smallest next_eligible
   2. If now < next_eligible → sleep until next_eligible
   3. Re-check current last_used_at (may have moved since the entry was inserted)
   4. If still stale → fire "threshold crossed" → push via notification hub
   5. If used again in the meantime → the reinsertion in step 1 already
      superseded this entry; nothing to do
```

This wakes up **exactly once per genuine threshold crossing**, never sweeping users who are actively using their apps. Shard the heap by `hash(user_id) % num_shards` to avoid a single-coordinator bottleneck — same rationale as the Crawler's frontier sharding (Section 16 of that guide).

### Duplicate-fire handling

If a fire and a fresh usage event race, the user might get a toast a few seconds after they opened the app. This is an idempotency/UX nit, not a correctness violation — dedup key `(user_id, app_id, threshold_window)` with a short TTL suppresses the obvious duplicate case without requiring strong coordination.

---

## 12. Component 6: Policy/Config Service

```
Config: {
  tenant_id:   "contoso.com" | "default",
  app_id:      "powerpoint",
  threshold_days: 14,
  updated_at:  timestamp
}
```

- Tenant admins can override the default threshold per app (e.g., a company that mandates PowerPoint literacy might set 7 days; a company that doesn't care about OneNote might set 90).
- Served through the same short-TTL cache as everything else in this system — a policy change propagating over ~15 minutes is a perfectly acceptable UX tradeoff for a nudge feature, and avoids needing synchronous global consistency for config changes.

---

## 13. Component 7: Caching Layer

### Why cache even absent a partition (PACELC "E" case)

Even with no network partition, there's a latency-vs-consistency choice on every read: hit the KV store directly (fresher, higher latency, more load) or serve from a per-user Redis cache with a short TTL (slightly stale, far cheaper, absorbs the ~74K req/sec read volume from Section 5).

Given the cost of a stale badge (someone sees an alert clear a few minutes later than technically possible) is negligible, and the cost of 74K+ req/sec hitting a primary store directly is real infrastructure spend, the **PACELC answer here is to favor Latency over Consistency even in the normal, non-partitioned case** — this is worth stating explicitly rather than only discussing CAP's partition scenario.

```
Cache key:   user_id
Cache value: { app_id: last_used_at, ... }  for all of that user's apps (one entry, not per-app)
TTL:         5-15 minutes
Invalidation: write-through on ingest (aggregator can optionally push a cache
              invalidation on write) OR simply let TTL expire — either is fine
              given the low staleness cost
```

Batching all of a user's apps into a single cache entry (rather than one key per app) also avoids N cache round-trips per taskbar refresh.

---

## 14. Cross-Device Reconciliation

A user might use PowerPoint on their desktop but not their laptop. The **badge must reflect usage across all their devices**, not just the local one.

```
Device A (just used PowerPoint):
  → local badge clears instantly (local-first, Section 7)
  → usage event uploaded async

Device B (hasn't used PowerPoint, same user):
  → still shows stale badge until:
    a) its next poll picks up the updated last_used_at from the server, or
    b) (Approach D) a push-hint tells it to refetch
  → this staleness window (minutes) is an accepted, explicit tradeoff —
    the requirement is "don't nag on the device where I just did the thing,"
    not "instantly sync my badge state across every device I own"
```

Worth stating this distinction explicitly in an interview: the two devices have **different correctness requirements** — the originating device needs sub-second suppression, other devices only need eventual consistency. Treating both the same way (e.g., waiting for a server round-trip everywhere) would over-engineer the common case and under-deliver on the one that actually matters for UX.

---

## 15. CAP / PACELC Theorem Positioning

| Component | CAP | PACELC | Reasoning |
|---|---|---|---|
| Usage ingestion (event log) | **AP** | PA / EL | At-least-once + idempotent max-merge; delayed events only delay cross-device sync, never corrupt state |
| last_used_at KV store | **AP** | PA / EL | Eventual consistency acceptable; reads can be a few minutes stale |
| Per-user Redis cache | **AP** | PA / EL | Deliberately favors latency over freshness even absent a partition — the defining PACELC choice of this system |
| Policy/Config service | **AP** (short-TTL cache) | PA / EL | Admin threshold changes propagate over minutes; no synchronous global consistency needed |
| Local device state | N/A — single-writer | N/A | No coordination needed; the device is authoritative for its own instant suppression |
| Delay-queue scheduler (push variant) | **AP** per shard | PA / EL | Sharded by user_id hash; duplicate/late fires are a UX nit, not a correctness break |

**No CP component appears anywhere in this design** — worth stating explicitly and contrasting against systems that do need one. The Web Crawler needs ZooKeeper/etcd (CP) for shard-assignment and leader election because the frontier has a cross-partition invariant (one FIFO queue per domain, exactly one owner). This system has no equivalent invariant: partitioning by `user_id` is handled natively by the KV store's own hashing, there's no uniqueness constraint, no leader election, and no distributed transaction anywhere in the write or read path. Recognizing when a coordinator is *not* needed is as much a Staff-level signal as knowing when one is.

---

## 16. Failure Modes & Mitigations

| Failure | Symptom | Detection | Mitigation |
|---|---|---|---|
| Client offline for days | Local device's own badge may be stale; usage not yet synced | Expected/benign | Local-first suppression is still correct for the device where usage actually happened; batch-upload on reconnect |
| Clock skew on client device | Locally-recorded "used just now" timestamp is wrong | Drift metrics vs. server ingestion time | Server-assigned timestamp is authoritative for cross-device merge; local clock only affects local instant-suppress UX, never correctness |
| Duplicate usage events | None — non-issue by design | N/A | Idempotent `max()` merge makes duplicates harmless |
| Login-hour thundering herd (e.g., 9am) | Query API / cache spike | Request rate per shard | Client-side poll jitter; per-user cache; exponential backoff |
| Missing telemetry (opt-out/compliance) | No last_used_at ever recorded for a user/app | Null state | **Fail-safe default: never alert on missing data** — see Section 17 |
| New install, never opened yet | False "unused" alert on day 1 | `last_used_at` null but `first_seen_at` recent | Threshold clock anchored to `first_seen_at`, not epoch/null (Section 10) |
| Policy propagation delay | Admin lowers threshold; some users briefly see old value | Cache TTL window | Bounded staleness (~15 min); acceptable for a non-critical policy |
| Duplicate push notifications (Approach C) | User sees the same toast twice | Delay-queue re-fire race | Idempotent dedup key `(user, app, threshold_window)` with short TTL |
| Shared/kiosk device | Usage attributed to the wrong user | Session/identity mismatch | Bind every usage event to the signed-in AAD/Entra identity, never the device |
| Hot key / hot partition | Effectively none | — | Partitioning by `user_id` is naturally uniform — unlike the Crawler's hot-domain problem, there's no adversarial or organically skewed key here |

---

## 17. Privacy, Compliance & Fail-Safe Defaults

Usage telemetry is sensitive, and enterprise tenants or individual users may disable it (GDPR, internal compliance policy, personal preference).

**The fail-safe direction matters:** when data is missing, the system must default to **never showing an alert**, not to showing one. An incorrectly-suppressed nag is a minor missed opportunity; an incorrectly-shown nag based on absent data (e.g., telling someone they haven't used an app when you simply have no visibility into their usage) is a false claim about their behavior and a worse failure mode. This mirrors the Web Crawler's robots.txt failure handling — "assume allowed" on ambiguity rather than the aggressive default.

Practical implications:
- Telemetry opt-out at the tenant or user level should short-circuit the Alert Query API to always return `alert: false` for that scope, rather than falling through to "no data → treat as very stale."
- Respect data residency requirements — the KV store and event log should support regional partitioning consistent with wherever the tenant's other O365 data resides.

---

## 18. Scalability & Sharding

| Component | Scaling strategy |
|---|---|
| Usage Ingestion API | Stateless; scale horizontally behind a load balancer |
| Kafka event log | Partition by `user_id`; add partitions as user base grows |
| Stream Aggregator | Stateless consumers, one group per partition set |
| last_used_at KV store | Native partitioning by `user_id`; managed auto-scaling (DynamoDB/Cosmos) or consistent hash ring (Cassandra) |
| Redis cache | Cluster mode, sharded by `user_id` |
| Delay-queue scheduler (if used) | Sharded by `hash(user_id) % num_shards`, mirroring the Crawler's frontier sharding — each shard owns its own heap, no cross-shard coordination required |

No component in this design requires a global coordinator for correctness (Section 15), which simplifies the scaling story considerably compared to the Crawler: there's no single frontier coordinator to shard away from in the first place.

---

## 19. Senior vs Staff Answer Differentiators

### Senior-level expectations

- Identify the need to track last-used timestamps per user/app
- Propose a database to store them
- Mention a cron job or scheduled check for "unused" detection
- Recognize the need to clear the alert when the app is used

### Staff-level expectations (additional)

| Topic | What Staff-level adds |
|---|---|
| Core mechanism | Alert as a **derived value computed at read time**, not a stored/precomputed flag — explicitly rejecting the sweep approach with a cost argument |
| Multiple approaches | Proactively compares badge (lazy) vs. toast (scheduled) as genuinely different problems with different architectures, not one design for both |
| Local-first design | Recognizes the "suppress instantly" requirement is best solved with zero server dependency, not lower server latency |
| Data store | Explicit KV schema, partitioned by `user_id`, sized and justified — mirrors known wide-row patterns (Crawler's `url_metadata`) |
| CAP/PACELC | Explicit per-component table, **including the PACELC argument for caching in the non-partitioned case**, and an explicit case for why no CP component is needed anywhere |
| Cross-device | Distinguishes the originating device's correctness requirement (instant) from other devices' (eventual) rather than treating all devices identically |
| Push extension | Reuses a min-heap/delay-queue pattern (explicitly cross-referenced to the Crawler's domain heap) instead of inventing a new scheduling primitive |
| Privacy | Explicit fail-safe direction: missing data → never alert, tied to compliance/opt-out, not just "handle privacy somehow" |
| New-install edge case | Anchors the threshold clock to `first_seen_at`, avoiding a false day-1 nag |

### The single most important Staff differentiator

**Recognizing which of the two problems you're actually solving** — "sync a piece of derived state to a badge" vs. "detect an event and push a notification" — and choosing the cheapest architecture for the one that's actually being asked, rather than defaulting to an event-driven scheduler because that's the more familiar notification-system pattern. Most candidates who've built notification systems before will over-apply that pattern here; the Staff-level move is recognizing this problem doesn't need it.

---

## 20. Interview Time Allocation

For a 45-minute system design session:

| Phase | Time | Focus |
|---|---|---|
| Requirements & clarifying questions | 5 min | Badge vs. toast, per-app thresholds, cross-device scope |
| Capacity estimation | 3 min | Rows, event throughput, read volume |
| High-level architecture (write + read path) | 7 min | Two-path diagram; name the durable log and the KV store |
| Local-first suppression (deep dive) | 5 min | Why zero-latency suppression must be local, not server-side |
| last_used_at store & lazy alert computation | 8 min | The core insight — derived state, no sweep |
| Push variant / delay-queue scheduler (if asked) | 6 min | Min-heap pattern, cross-reference to prior designs if relevant |
| CAP/PACELC per component | 5 min | Including the "no CP needed" argument |
| Failure modes & privacy fail-safe | 4 min | Missing-data direction, new-install edge case |
| Wrap-up | 2 min | Cheatsheet-level summary |

**What to cut if short on time:** the push/delay-queue extension (Section 11) — it's an extension, not the core ask. **Never cut:** the lazy-vs-sweep framing (Section 10) — it's the answer to the actual question being asked.

---

## 21. Quick-Reference Cheatsheet

```
THE CORE INSIGHT
────────────────
alert = (now - last_used_at) > threshold
Computed at READ time. No sweep. No precomputed flag. No scheduler
needed for the badge case.

TWO DIFFERENT PROBLEMS, TWO DIFFERENT DESIGNS
──────────────────────────────────────────────
Persistent badge:        Lazy read-time computation (Approach B)
Proactive toast/push:    Min-heap delay-queue, keyed on next_eligible
                          (Approach C — same pattern as Crawler's
                          domain heap keyed on next_ok_ts)

LOCAL-FIRST SUPPRESSION
────────────────────────
"Used just now" → clear badge LOCALLY, zero server round-trip.
Never let the hardest latency requirement depend on the slowest path.

KEY NUMBERS (illustrative)
───────────────────────────
400M users x 25 apps  →  10B (user,app) rows  →  ~600GB total state
8B usage events/day   →  ~93K events/sec avg, ~740K/sec peak
6.4B alert queries/day →  ~74K req/sec avg  →  absorbed mostly by cache

KEY SYSTEMS
───────────
Event log:        Kafka, partitioned by user_id
State store:       DynamoDB/Cosmos/Cassandra, partition key = user_id
Cache:              Redis, per-user entry, 5-15 min TTL
Coordination:       NONE NEEDED — no CP component anywhere in this design

CAP DECISIONS
─────────────
AP everywhere: ingestion, state store, cache, policy service, scheduler shards
CP: none — explicitly justified by absence of cross-partition invariants

FAIL-SAFE DIRECTION
────────────────────
Missing/opted-out telemetry → NEVER alert (not "always alert")

STAFF-LEVEL SIGNALS TO HIT
───────────────────────────
✓ Derived state vs. precomputed flag — the headline distinction
✓ Multiple approaches compared with explicit pros/cons before committing
✓ Local-first design for the latency-critical suppression path
✓ Explicit PACELC argument for caching even absent a partition
✓ Explicit "no CP needed here" — knowing when NOT to add a coordinator
✓ Cross-device correctness split: instant on origin device, eventual elsewhere
✓ Delay-queue/min-heap reuse for the push extension, not a new primitive
✓ Fail-safe default direction on missing privacy-sensitive data
✓ New-install grace period anchored to first_seen_at
```

---

*Guide compiled for Senior & Staff-level system design interview preparation.*
*Topics: Notification Systems, State Sync, Lazy Evaluation, CAP/PACELC, Cross-Device Consistency.*
*Cross-references: Web Crawler (Domain Heap / min-heap-by-next-eligible-time pattern, url_metadata partitioning), Office 365 Subscription Renewal Notification System, general Notification System guide.*
