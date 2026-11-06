# Ticketmaster — System Design Interview Guide
### Senior & Staff Engineer Level

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
9. [Deep Dive Q1: Viewing Event Details](#9-deep-dive-q1-viewing-event-details)
10. [Deep Dive Q2: Searching for Events](#10-deep-dive-q2-searching-for-events)
11. [Deep Dive Q3: Booking Specific Seats](#11-deep-dive-q3-booking-specific-seats)
12. [Deep Dive Q4: Two-Phase Booking — Never Lose a Seat at Checkout](#12-deep-dive-q4-two-phase-booking--never-lose-a-seat-at-checkout)
13. [Deep Dive Q5: Scaling Reads to 10M Concurrent Users](#13-deep-dive-q5-scaling-reads-to-10m-concurrent-users)
14. [Deep Dive Q6: Handling Complex Search Queries](#14-deep-dive-q6-handling-complex-search-queries)
15. [Deep Dive Q7: Real-Time Seat Map](#15-deep-dive-q7-real-time-seat-map)
16. [Deep Dive Q8: Virtual Waiting Room](#16-deep-dive-q8-virtual-waiting-room)
17. [CAP / PACELC Positioning](#17-cap--pacelc-positioning)
18. [Failure Modes & Mitigations](#18-failure-modes--mitigations)
19. [Scalability & Sharding](#19-scalability--sharding)
20. [Senior vs Staff Answer Differentiators](#20-senior-vs-staff-answer-differentiators)
21. [Interview Time Allocation](#21-interview-time-allocation)
22. [Quick-Reference Cheatsheet](#22-quick-reference-cheatsheet)
23. [Related Guides](#23-related-guides)

---

## 1. Problem Statement & Scope

Ticketmaster is a marketplace where users **discover** live events (concerts, sports, theatre), **inspect** a venue seat map, and **buy tickets for specific physical seats**. Every seat for an event is exactly one ticket — it can be sold **once and only once**.

### The anchor constraint

Two radically different workloads share one product:

| Workload | Shape | What matters |
|---|---|---|
| **Browse / search / view** | ~100:1 to 1000:1 read-heavy, cacheable, spiky | Availability, latency, cost |
| **Reserve / book** | Low average volume, *extreme* contention on one event at on-sale time | Correctness: **zero double-sells** |

Every downstream decision follows from this split: the read path is **AP/EL** and leans on caches, CDN and eventual consistency; the inventory path is a small **CP island** with a single source of truth per event.

### What the interviewer is really testing

- Can you guarantee **exactly-once seat ownership** under massive concurrency without a global lock?
- Do you understand **temporary holds** — TTLs, expiry, and the payment-after-expiry race?
- Can you scale a **hot key** (one Taylor-Swift-sized event) rather than just "add more servers"?
- Do you treat **fairness** (waiting room, bots) as an engineering requirement, not a product footnote?

### Out of scope

General admission / section-based inventory, resale marketplace (Ticketmaster Resale / StubHub-style), dynamic pricing algorithm internals, event-creation admin tooling, ticket transfer, barcode entry scanning.

---

## 2. Functional Requirements

### Full list

| # | Requirement | Priority |
|---|---|---|
| FR1 | View event details (name, description, venue, date, performers, price range) | **Core** |
| FR2 | Search events by keyword, date range, location, category | **Core** |
| FR3 | Reserve (temporarily hold) specific seats | **Core** |
| FR4 | Confirm booking — pay and finalize within the hold window | **Core** |
| FR5 | View seat map with live availability | Extended (Q7) |
| FR6 | Virtual waiting room for high-demand events | Extended (Q8) |
| FR7 | Receive confirmation / e-ticket (email, app wallet) | Supporting |
| FR8 | View "My Tickets" order history | Supporting |

### The four core FRs in detail

#### FR1 — View event details

When a user clicks an event, they see **static-ish metadata**: name, description, performer(s), venue name and address, date/time, images, price range, and a seat map.

Key design insight: an event page is really **two documents with different change rates**:

| Part | Changes | Freshness need | Strategy |
|---|---|---|---|
| Event metadata (name, venue, date, images, seat *layout*) | Rarely (hours/days) | Minutes of staleness OK | Aggressively cached (CDN + Redis) |
| Seat *availability* | Every second during on-sale | Seconds, and only a *hint* | Separate endpoint / push channel |

Splitting them lets 99% of the page be served from the CDN while the volatile part is small and targeted. Bundling them into one response is a classic Senior slip — it forces the whole page to have a 1-second TTL.

#### FR2 — Search events

Users search by **free text** ("taylor swift", "lakers"), filtered by **date range**, **location** (city or "near me" radius), **category/genre**, and optionally **price**. Results must support **relevance ranking**, **pagination**, and tolerate **typos** ("tayler swft").

Implications:
- Free-text + typo tolerance + geo + facets → an inverted index (Elasticsearch/OpenSearch), not SQL `LIKE`.
- Search is **eventually consistent** with the source of truth; a newly created event appearing 5 seconds later is fine.
- "Sold out" status should be reflected in search, but seconds of lag is acceptable.

#### FR3 — Reserve specific seats (Phase 1)

A user selects 1–N specific seats (e.g., Section 104, Row G, Seats 7–8) and clicks "Reserve". The system must:

1. **Atomically** move *all* selected seats from `AVAILABLE` → `RESERVED` — **all-or-nothing** (getting seat 7 but not 8 is a failure for a couple).
2. Associate the hold with the user's pending booking and an **expiry** (e.g., 10 minutes).
3. Make those seats invisible/unselectable to everyone else for the hold duration.
4. Release them automatically if the user abandons checkout.

This is where the double-sell guarantee lives.

#### FR4 — Confirm booking (Phase 2)

Within the hold window, the user pays. On successful payment the system:

1. Moves seats `RESERVED` → `BOOKED` and booking `PENDING` → `CONFIRMED`.
2. Must **never charge a user for a seat they don't own**, and **never let a paying user lose the seat** because of a timing race at the expiry boundary.
3. Must be **idempotent**: double-clicking "Pay", retried requests, and duplicate payment-provider webhooks must produce exactly one charge and one booking.
4. Emits a `BookingConfirmed` event → e-ticket issuance, email, analytics.

---

## 3. Non-Functional Requirements

### Full list

| # | Requirement | Target (illustrative) |
|---|---|---|
| NFR1 | **Strong consistency for inventory** — no seat sold twice | 0 double-sells, ever |
| NFR2 | **High availability & low latency for reads/search** | 99.99% availability; p99 < 200ms view, < 500ms search |
| NFR3 | **Scalability under bursty load** | 10M concurrent users on a single on-sale; 100× normal traffic in seconds |
| NFR4 | **Fairness & abuse resistance** | Arrival-order (or lottery) fairness; bots cannot crowd out humans |
| NFR5 | Durability of bookings & payments | No confirmed booking ever lost (RPO = 0 for orders) |
| NFR6 | Security / compliance | PCI-DSS: never store raw card data (tokenize via payment provider) |
| NFR7 | Observability | Per-event dashboards: hold rate, conversion, queue depth, double-sell canary |

### The four core NFRs in detail

#### NFR1 — Strong consistency for seat inventory (CP)

A seat can belong to at most one booking at any instant. Under partition or failure, the inventory path **refuses writes** rather than risk a double-sell — a refund, an angry customer at the venue gate, and a press story are far more expensive than a "please try again" error.

How it's achieved (detailed in Q3/Q4):
- **Single source of truth** per event: one Postgres primary owns all tickets for an event (sharded by `event_id`), so every seat transition is a single-shard ACID transaction — no 2PC.
- **Conditional atomic updates** (`UPDATE … WHERE status = 'AVAILABLE'`) act as compare-and-swap.
- A **unique constraint** on `(ticket_id)` in the confirmed-ticket ledger is the last line of defense.
- Caches, Redis locks, and the seat map are **hints**; they can be wrong without causing a double-sell.

#### NFR2 — High availability & low latency for reads (AP/EL)

Browsing is where 99%+ of traffic lives. A user seeing an event description that's 60 seconds stale costs nothing; a user seeing an error page costs revenue and trust.

- Event metadata: CDN + Redis, stale-while-revalidate, served even if the database is down.
- Search: Elasticsearch replicas; queries never touch the primary DB.
- Seat availability: served from a precomputed projection, refreshed in near-real-time, never from `SELECT` on the tickets table at scale.
- PACELC framing: in normal operation (no partition) we still choose **Latency over Consistency** for reads.

#### NFR3 — Scalability for extreme, bursty demand

Ticketmaster's traffic isn't high-average; it's **spiky and concentrated**. A mega on-sale can bring millions of users to *one event* in the first 60 seconds — a **hot key** problem, not a horizontal-sharding problem. Adding shards doesn't help when all demand targets a single event that lives on one shard.

Techniques (Q5, Q8):
- **Admission control** (virtual waiting room) caps the write path to what inventory and DB can absorb.
- **Hot-key mitigation** on the read path: L1 in-process caches, key replication, request coalescing.
- **Stateless services** autoscaled — and *pre-scaled* ahead of announced on-sales (on-sale times are known in advance — a Staff point: use the schedule, don't rely on reactive autoscaling that lags by minutes).

#### NFR4 — Fairness & bot resistance

With 2M people competing for 70K seats, the *distribution* of who gets tickets is a product promise. Without engineering for it, scalpers' bots — faster, parallel, scripted — win.

- **Virtual waiting room** with deterministic ordering (arrival time or randomized lottery for pre-on-sale arrivals).
- **Signed admission tokens** gate the seat-selection and reserve endpoints; no token, no seat map.
- **Per-user hold limits** (e.g., max 8 seats, one active hold per event).
- **Bot friction**: CAPTCHA / proof-of-work at queue join, device fingerprinting, verified-fan pre-registration, rate limits per account/IP/ASN.

---

## 4. Capacity Estimation

> All numbers are **illustrative** (SI/decimal units). State them as assumptions in the interview.

```
USERS & TRAFFIC
───────────────
Daily active users (normal):        ~10M
Peak concurrent (mega on-sale):     10M on the platform, ~2–3M on ONE event

Read RPS (peak):
  10M users × 1 request / 10s       ≈ 1M RPS (client-facing)
  CDN hit ratio ~90%                → ~100K RPS reach origin
  Redis serves ~95% of origin reads → ~5K RPS reach the DB (replicas)

Search RPS (normal): ~10K QPS; peak ~50K QPS

BOOKINGS
────────
Tickets sold per year:              ~500M  (illustrative)
Average:                            500M / 31.5M s ≈ 16 tickets/s
Mega on-sale: 70K-seat stadium sold in ~15 min
  Confirms:                         70,000 / 900s ≈ 80 bookings/s
  Reserve ATTEMPTS without queue:   2M users × several clicks → 100K+ writes/s on ONE shard ✗
  Reserve attempts WITH queue:      admit ~2K users/min → ~50–100 writes/s ✓
→ The waiting room turns an impossible write load into a trivial one.

STORAGE
───────
Events:   1M events/year × 5KB metadata      ≈ 5 GB/year     (fits in RAM/cache)
Tickets:  1M events × avg 5K seats = 5B rows × ~200B ≈ 1 TB/year
          → shard by event_id; archive events > 1 year to cold storage
Bookings: ~200M orders/year × 1KB            ≈ 200 GB/year

SEAT MAP PAYLOAD (per event, 70K seats)
──────────────────────────────────────
Naive JSON (70K × ~60B)            ≈ 4.2 MB  ✗  × 1M viewers = 4.2 TB of egress
Split:
  Static layout (coords, labels)   ≈ 1–2 MB, immutable → CDN, cached forever (versioned URL)
  Availability bitmap (2 bits/seat)≈ 17.5 KB → ~5 KB gzipped  ✓

REAL-TIME FAN-OUT (Q7)
──────────────────────
Seat changes during on-sale: 70K seats × ~2 transitions / 900s ≈ 150 changes/s
Batched into 1 delta/s per event
Subscribers: 1M viewers × 1 msg/s ≈ 1M messages/s, ~300B each ≈ 300 MB/s egress
SSE nodes at 50K connections each → 20 nodes minimum → provision ~60 (3× headroom)

WAITING ROOM (Q8)
─────────────────
Queue size:   up to ~14M entries × ~100B ≈ 1.4 GB in Redis — fine
Join burst:   ~1M joins in the first few seconds → single-key INCR (~100K ops/s) is a HOT KEY
              → edge-stamped signed arrival tokens (no central counter on join)
```

**Key insight:** Average write volume is trivially small (~16/s). The entire challenge is **concentration**: millions of readers and thousands of writers on a *single event* in a *single minute*. Design for the hot event, not the average one.

---

## 5. Core Entities

```
┌──────────────┐     ┌──────────────┐     ┌──────────────────┐     ┌──────────────┐
│   Venue      │1   *│   Event      │1   *│ EventPerformer   │*   1│  Performer   │
│──────────────│─────│──────────────│─────│──────────────────│─────│──────────────│
│ venue_id     │     │ event_id     │     │ event_id     (PK)│     │ performer_id │
│ name         │     │ venue_id     │     │ performer_id (PK)│     │ name         │
│ address, geo │     │ name, desc   │     └──────────────────┘     │ genre        │
│ layout_id ───┼──┐  │ start_time   │                              └──────────────┘
└──────────────┘  │  │ category     │
                  │  │ status       │ (ON_SALE, SOLD_OUT, CANCELLED)
                  │  │ queue_enabled│
                  │  └──────┬───────┘
                  │         │1
┌─────────────────▼┐        │*
│  SeatLayout/Seat │   ┌────▼──────────────────────────┐
│──────────────────│   │   Ticket (one per seat/event) │
│ seat_id          │1 *│───────────────────────────────│
│ section,row,num  │───│ ticket_id                     │
│ x, y (map coords)│   │ event_id   (SHARD KEY)        │
└──────────────────┘   │ seat_id                       │
                       │ price                         │
                       │ status  AVAILABLE|RESERVED|   │
                       │         BOOKED                │
                       │ booking_id (nullable)         │
                       │ hold_expires_at (nullable)    │
                       │ version                       │
                       └────┬──────────────────────────┘
                            │*
                            │1
┌──────────────┐1      *┌───▼──────────────────────────┐
│    User      │───────│   Booking                     │
│──────────────│       │───────────────────────────────│
│ user_id      │       │ booking_id                    │
│ email        │       │ user_id, event_id             │
│ verified_fan │       │ status PENDING|PAYMENT_PENDING│
└──────────────┘       │        |CONFIRMED|EXPIRED|    │
                       │        CANCELLED|REFUNDED     │
                       │ total_amount (server-computed)│
                       │ payment_intent_id             │
                       │ idempotency_key (UNIQUE)      │
                       │ expires_at, created_at        │
                       └───────────────────────────────┘

Waiting-room entities (Redis, ephemeral):
  QueueEntry      {event_id, user_id, queue_number, joined_at}
  AdmissionToken  signed JWT {event_id, user_id, device_id, exp, jti}
```

### Why these modeling choices

- **`EventPerformer` is a join table** on `(event_id, performer_id)` — Event↔Performer is many-to-many (a tour reuses performers across events; a festival event has multiple performers), so it can't be a plain FK on either side.
- **Ticket is a first-class row per (event, seat)** — not a counter. Seat-level booking requires seat-level state; the row *is* the lock target.
- **Seat layout is per venue, ticket is per event.** The physical layout is reused across hundreds of events and cached forever; only ticket rows are per-event.
- **`hold_expires_at` on the ticket** enables *lazy expiry*: an expired hold is treated as available at read/write time — no sweeper needed for correctness.
- **`version`** supports optimistic concurrency and lets the seat-map projection order updates.
- **`idempotency_key` UNIQUE on Booking** collapses retries / double-clicks into one booking.
- **`total_amount` is computed server-side** from ticket prices — never trusted from the client.

---

## 6. API / System Interface

All endpoints are behind an API gateway with JWT auth. **`userId` always comes from the token, never from the request body.**

### Read APIs

```http
GET /v1/events/{eventId}
→ 200 { eventId, name, description, startTime, category,
        venue: { venueId, name, address, geo, layoutUrl },
        performers: [...], priceRange: {min,max}, status, queueEnabled }
  Cache-Control: public, max-age=60, stale-while-revalidate=300

GET /v1/events/{eventId}/availability
→ 200 { eventId, version, encoding: "2bit-bitmap", data: "<base64>",
        sectionSummary: [{section, available}] }
  Requires: Admission-Token header when event.queueEnabled

GET /v1/events/{eventId}/availability/stream          (SSE)
→ text/event-stream
  event: delta   data: { fromVersion, toVersion, changes: [[ticketIdx, status], ...] }
  Supports Last-Event-ID resume

GET /v1/search?q=taylor+swift&city=seattle&lat=47.6&lng=-122.3&radius=50km
               &from=2026-11-01&to=2026-12-31&category=concert
               &sort=relevance|date|distance&cursor=<opaque>&limit=20
→ 200 { results: [EventSummary], nextCursor, facets: { category:{...}, city:{...} } }
```

### Booking APIs (two-phase)

```http
POST /v1/bookings/reserve
Headers: Idempotency-Key: <uuid>, Admission-Token: <jwt>
Body:    { eventId, ticketIds: ["t_104_G_7", "t_104_G_8"] }
→ 201 { bookingId, status: "PENDING", expiresAt, tickets:[...], totalAmount }
→ 409 { error: "SEATS_UNAVAILABLE", unavailable: ["t_104_G_8"] }
→ 403 { error: "ADMISSION_REQUIRED" }

POST /v1/bookings/{bookingId}/checkout
→ 200 { paymentClientSecret, expiresAt }       // creates Stripe PaymentIntent,
                                               // moves booking → PAYMENT_PENDING

POST /v1/webhooks/payments                     // from payment provider (signed)
Body: { type: "payment_intent.succeeded", paymentIntentId, metadata:{bookingId} }

GET  /v1/bookings/{bookingId}                  // client polls / resumes after reload
→ 200 { bookingId, status, expiresAt, tickets }

DELETE /v1/bookings/{bookingId}                // user explicitly releases hold
```

### Waiting room APIs

```http
POST /v1/queue/{eventId}/join          (CAPTCHA/PoW token required)
→ 202 { queueTicket: <signed>, position, etaSeconds }

GET  /v1/queue/{eventId}/status        (or SSE /status/stream)
Headers: Queue-Ticket: <signed>
→ 200 { position, etaSeconds, state: "WAITING" }
→ 200 { state: "ADMITTED", admissionToken: <jwt>, admissionExpiresAt }
→ 200 { state: "SOLD_OUT" }
```

### API design notes (Staff signals)

- **Why `checkout` is a separate call from `reserve`:** it's the moment we extend the hold into the payment grace window (Q4) — the explicit state transition is what prevents losing seats mid-payment.
- **Why the confirm happens via webhook, not client callback:** the client can close the tab after paying; the provider's signed webhook is the authoritative payment signal.
- **Cursor pagination for search**, not offset: deep offsets are expensive in Elasticsearch (`search_after`) and unstable under index updates.

---

## 7. Data Flow

### Flow A — View event
```
Client → CDN (hit? return) → Gateway → Event Service
       → Redis (hit? return) → Postgres read replica → populate Redis → return
Client → CDN: layoutUrl (immutable, versioned)  → seat coordinates
Client → Seat-Map Service: availability snapshot + SSE stream
```

### Flow B — Search
```
Client → CDN (popular normalized queries cached ~30s)
       → Gateway → Search Service → Elasticsearch (replica shards) → ranked hits
Indexing path (async):
Postgres (events) → WAL → Debezium CDC → Kafka [event-changes] → Indexer → Elasticsearch
```

### Flow C — Reserve (Phase 1)
```
Client → Gateway (validate JWT + Admission-Token, rate limit)
       → Booking Service
           1. Idempotency check (booking with this key exists? return it)
           2. BEGIN on event's shard
              UPDATE tickets SET RESERVED … WHERE AVAILABLE or hold expired
              rowcount == requested?  else ROLLBACK → 409
              INSERT booking (PENDING, expires_at)
              INSERT outbox (SeatsReserved)
              COMMIT
       → 201 {bookingId, expiresAt}
Outbox relay → Kafka [seat-changes] → Seat-Map Fan-out → SSE → all viewers
```

### Flow D — Confirm (Phase 2)
```
Client → POST /checkout → Booking Service
           booking PENDING → PAYMENT_PENDING, extend hold by grace (atomic, only if not expired)
           create PaymentIntent (idempotency key = bookingId)
Client → Payment provider (card entry; PCI scope stays with provider)
Provider → POST /webhooks/payments (signed, at-least-once)
       → Booking Service: tickets RESERVED → BOOKED, booking → CONFIRMED (idempotent)
       → outbox: BookingConfirmed, SeatsBooked
Kafka → Notification Service (email/e-ticket), Seat-Map Fan-out, Analytics, Search (sold-out flag)
```

### Flow E — Hold expiry (abandoned checkout)
```
Correctness: lazy — any reserve sees hold_expires_at < now() as AVAILABLE. No job needed.
Hygiene:     Expiry scheduler (Redis ZSET of expiries / delayed queue)
             → at expires_at: UPDATE booking → EXPIRED, tickets → AVAILABLE (conditional)
             → outbox: SeatsReleased → seat map shows them green again
```

---

## 8. High-Level Design

```
                                   ┌────────────────────────┐
                                   │   Clients (Web / iOS / │
                                   │   Android)             │
                                   └───────────┬────────────┘
                                               │
                           ┌───────────────────▼─────────────────────┐
                           │  CDN / Edge (static assets, event pages,│
                           │  seat layouts, waiting-room page,       │
                           │  edge-stamped queue arrival tokens, WAF)│
                           └───────────────────┬─────────────────────┘
                                               │
                           ┌───────────────────▼─────────────────────┐
                           │  API Gateway                            │
                           │  JWT auth · rate limiting · bot checks  │
                           │  Admission-Token validation (stateless) │
                           └──┬───────┬────────┬────────┬────────┬───┘
                              │       │        │        │        │
          ┌───────────────────┘       │        │        │        └──────────────────┐
          │                ┌──────────┘        │        └──────────┐                │
┌─────────▼────────┐ ┌─────▼──────────┐ ┌──────▼────────┐ ┌────────▼────────┐ ┌─────▼──────────┐
│  Event Service   │ │ Search Service │ │Booking Service│ │ Seat-Map Service│ │ Waiting Room   │
│  (FR1)           │ │ (FR2)          │ │ (FR3, FR4)    │ │ snapshot + SSE  │ │ Service (FR6)  │
│  stateless       │ │ stateless      │ │ stateless     │ │ fan-out (FR5)   │ │ + Admission    │
└───┬─────────┬────┘ └──────┬─────────┘ └──┬───────┬────┘ └───┬────────▲────┘ │   Controller   │
    │         │             │              │       │          │        │      └──────┬─────────┘
    │   ┌─────▼─────────────┼──────┐       │       │          │        │             │
    │   │ Redis Cluster (cache)    │◄──────┼───────┼──────────┘        │      ┌──────▼─────────┐
    │   │ event:{id}, seatmap:{id} │       │       │                   │      │ Redis (queue)  │
    │   └──────────────────────────┘       │       │                   │      │ counters, ZSET │
    │                                      │       │                   │      └────────────────┘
┌───▼──────────────────────────┐   ┌───────▼───────────────────────┐   │
│ Postgres — Events/Venues     │   │ Postgres — Tickets & Bookings │   │
│ (small; primary + replicas)  │   │ SHARDED by event_id           │   │
└───┬──────────────────────────┘   │ primary + sync standby / shard│   │
    │ WAL / CDC                    │ + outbox table                │   │
    │                              └───────┬───────────────────────┘   │
    │                                      │ outbox relay / CDC        │
┌───▼──────────────────────────────────────▼─────────────────────┐     │
│                  Kafka                                         │     │
│  topics: event-changes · seat-changes (key=event_id) ·         │─────┘
│          booking-events                                        │
└───┬───────────────────────────┬──────────────────────┬─────────┘
    │                           │                      │
┌───▼──────────────┐  ┌─────────▼──────────┐  ┌────────▼──────────┐
│ Search Indexer   │  │ Notification Svc   │  │ Analytics / Fraud │
│ → Elasticsearch  │  │ email, e-ticket,   │  │ (ClickHouse)      │
│   (Search Svc    │  │ wallet pass        │  └───────────────────┘
│    reads here)   │  └────────────────────┘
└──────────────────┘
                                   ┌──────────────────────┐
     Booking Service ◄────────────►│ Payment Provider     │
     (PaymentIntent + webhooks)    │ (Stripe/Adyen) — PCI │
                                   └──────────────────────┘
```

### How each functional requirement is satisfied

| FR | Path through the design |
|---|---|
| FR1 View event | CDN → Event Service → Redis → Postgres replica. Seat layout from CDN (immutable). |
| FR2 Search | Search Service → Elasticsearch, kept in sync by CDC → Kafka → Indexer. |
| FR3 Reserve | Gateway (admission check) → Booking Service → single-shard conditional UPDATE in tickets DB. |
| FR4 Confirm | Booking Service ↔ Payment provider; webhook finalizes `RESERVED → BOOKED`. |
| FR5 Live seat map | Outbox → Kafka `seat-changes` → Seat-Map Service → SSE deltas to viewers. |
| FR6 Waiting room | Edge serves queue page → Waiting Room Service assigns order → Admission Controller issues tokens validated at Gateway. |
| FR7 Confirmation | Kafka `booking-events` → Notification Service. |

### Why each major component exists

| Component | Why |
|---|---|
| **CDN / Edge** | Absorbs ~90% of read traffic; serves the waiting-room page so origin is protected even under a stampede. |
| **API Gateway** | One place for auth, rate limits, and stateless admission-token checks — keeps the queue enforcement off the services' hot paths. |
| **Separate Booking Service** | Isolates the CP, contention-heavy path; scaled and deployed independently; small blast radius. |
| **Postgres sharded by `event_id`** | ACID + row locks + conditional updates; all seats of an event on one shard → single-node transactions, no 2PC. |
| **Transactional outbox** | Seat-state DB change and the event that announces it commit atomically — no "DB updated but Kafka publish lost" dual-write bug. |
| **Kafka** | Durable fan-out to seat map, search, notifications, analytics; replay; per-event ordering via `key=event_id`. |
| **Elasticsearch** | Inverted index, fuzzy matching, geo queries, facets — none of which a relational DB does well at scale. |
| **Redis (cache)** | Sub-ms reads for event metadata and availability snapshots. |
| **Redis (queue)** | Fast ordered structures for waiting-room position and admission cursors. |

### Database choice — options considered

| Option | Pros | Cons | Verdict |
|---|---|---|---|
| **Postgres (sharded by event_id)** | ACID, row-level locks, `UPDATE…WHERE` CAS, unique constraints, mature | Manual sharding (or Citus); single-primary write ceiling per shard | ✅ Tickets/Bookings |
| DynamoDB | Managed, infinite scale, conditional writes (`ConditionExpression`), `TransactWriteItems` (≤100 items) | Hot partition on mega-event; transactions cost 2× WCU; weaker ad-hoc queries | ✅ Viable alternative — mention it |
| Cassandra | Massive write throughput, AP | LWT (Paxos) is slow and contention-hostile; multi-row atomicity across seats is awkward | ❌ Wrong fit for inventory |
| MongoDB | Flexible docs | Multi-doc transactions exist but contention behavior worse than Postgres | ⚠️ Possible, not preferred |
| Spanner / CockroachDB | Global strong consistency, automatic sharding | Cost; cross-region latency on every write; overkill since an event's inventory is naturally single-region | ⚠️ Overkill |

---

## 9. Deep Dive Q1: Viewing Event Details

> **Q1. How will users be able to view simple event details when clicking on an event — name, description, location, date, etc.?**

### Answer

`GET /v1/events/{eventId}` hits the **Event Service**, which returns a denormalized `EventDetail` document assembled from the `events`, `venues`, and `performers` tables. Because this data changes rarely and is read constantly, it is served through a **three-layer cache hierarchy**: CDN → Redis → Postgres read replica.

```
Client ──▶ CDN (edge) ──hit──▶ return (≈90%)
              │ miss
              ▼
          API Gateway ──▶ Event Service
                              │
                              ├─▶ L1 in-process cache (1–5s TTL)  — hot events only
                              ├─▶ Redis  event:{id}  (TTL 10 min + CDC invalidation)
                              └─▶ Postgres read replica  (JOIN events+venue+performers)
```

### Options considered

| Option | Pros | Cons |
|---|---|---|
| **A. Query Postgres primary per request** | Simplest; always fresh | Primary becomes a bottleneck; 3-way JOIN on every click; doesn't survive an on-sale spike |
| **B. Read replicas** | Offloads primary; still SQL | Replica lag (seconds); still per-request JOIN cost; scaling replicas is coarse |
| **C. Cache-aside in Redis** | Sub-ms; cuts DB load 95%+ | Invalidation needed; stampede risk on expiry of a hot key |
| **D. Precomputed denormalized document** (written on event update) | Zero JOINs on read; one key fetch | Write amplification when a venue/performer changes (fan-out to all its events) |
| **E. CDN caching of the API response** | Absorbs the bulk of traffic at the edge; global latency | Needs explicit purge on change; must not cache personalized fields |

**Recommended:** **C + D + E** — a precomputed `EventDetail` document stored in Redis (rebuilt by a CDC consumer when any source row changes), fronted by the CDN with `max-age=60, stale-while-revalidate=300`, with read replicas as the fallback on cache miss.

### Key design points

1. **Separate static from dynamic.** The event detail response contains **no live seat availability** — only `priceRange` and `status`. Availability comes from a different endpoint (Q7). This is what makes the page cacheable at all.
2. **Seat layout is immutable and versioned.** `layoutUrl = /layouts/{venueId}/v{n}.json` → `Cache-Control: immutable, max-age=31536000`. A venue reconfiguration produces a new version URL; nothing needs purging.
3. **Invalidation via CDC, not via application code.** Postgres WAL → Debezium → Kafka `event-changes` → a consumer that rebuilds `event:{id}` in Redis and issues a CDN purge for `/v1/events/{id}`. Application writers can't "forget" to invalidate.
4. **Personalization stays out of the cached response.** "You have 2 tickets to this event" is a separate, uncached, per-user call.
5. **Graceful degradation.** If Postgres is down, Redis + CDN `stale-if-error` keep event pages up — reads are AP.

---

## 10. Deep Dive Q2: Searching for Events

> **Q2. How will users be able to search for events?**

### Answer

A dedicated **Search Service** queries **Elasticsearch**, which holds a denormalized `event` document per event (event + venue + performer fields flattened). The index is populated **asynchronously** from Postgres via **CDC → Kafka → Indexer**, so the transactional DB is never queried for search.

```
WRITE PATH (async, seconds of lag)
Postgres events/venues/performers ──WAL──▶ Debezium ──▶ Kafka [event-changes]
                                                          │
                                                          ▼
                                              Indexer (joins + denormalizes)
                                                          │
                                                          ▼
                                                   Elasticsearch

READ PATH
Client ──▶ Gateway ──▶ Search Service ──▶ Elasticsearch (replicas) ──▶ ranked results
```

### Options considered

| Option | How | Pros | Cons |
|---|---|---|---|
| **A. SQL `LIKE '%term%'`** | `WHERE name LIKE '%swift%'` | Zero new infra | Leading wildcard = **full table scan**; no relevance ranking; no typo tolerance; melts the DB at scale |
| **B. Postgres full-text search** | `tsvector` column + GIN index; `PostGIS` for geo | Single system; transactional consistency; decent for moderate scale | Weaker relevance tuning, fuzzy matching, and faceting; search load competes with booking-critical DB |
| **C. Elasticsearch / OpenSearch** | Inverted index, BM25, `geo_distance`, aggregations | Built for this: fuzzy, synonyms, facets, geo, boosting, horizontally scalable | Another system to operate; eventual consistency; sync pipeline needed |
| **D. Managed search (Algolia / Typesense)** | Hosted index | Great typeahead UX, low ops | Cost at scale; less control; data leaves your infra |

**Recommended:** **C**. For an early-stage MVP, **B** is a legitimate starting point — say so, and name the trigger to migrate (search QPS competing with booking writes, or product need for typo tolerance/facets).

### Index document

```json
{
  "event_id": "e_123",
  "name": "Taylor Swift | The Eras Tour",
  "description": "...",
  "performers": ["Taylor Swift"],
  "category": "concert",
  "genre": ["pop"],
  "venue_name": "Lumen Field",
  "city": "Seattle",
  "location": { "lat": 47.595, "lon": -122.332 },
  "start_time": "2026-12-05T19:00:00-08:00",
  "min_price": 89.0,
  "status": "ON_SALE",
  "popularity": 0.98
}
```

### Example query (keyword + date + geo)

```json
{
  "query": {
    "bool": {
      "must":   [{ "multi_match": { "query": "taylor swift",
                    "fields": ["name^3", "performers^4", "venue_name", "description"],
                    "fuzziness": "AUTO" } }],
      "filter": [
        { "range": { "start_time": { "gte": "2026-11-01", "lte": "2026-12-31" } } },
        { "geo_distance": { "distance": "50km", "location": { "lat": 47.6, "lon": -122.3 } } },
        { "term": { "status": "ON_SALE" } }
      ]
    }
  },
  "sort": ["_score", { "start_time": "asc" }],
  "search_after": ["<cursor from previous page>"],
  "size": 20
}
```

Filters go in `filter` context (cacheable bitsets, no scoring cost); free text goes in `must` (scored).

---

## 11. Deep Dive Q3: Booking Specific Seats

> **Q3. How will users be able to book specific seats for events? Each physical seat has exactly one ticket. Do not consider General Admission or section-based booking.**

### Answer

### Ticket provisioning (precondition, out of scope)

Booking never `INSERT`s a ticket row — the entire reserve transaction below is a pure `UPDATE`. That only works because every seat's `Ticket` row must already exist in `AVAILABLE` state *before* the event goes on sale. This design assumes a one-time, offline provisioning step: when an event is created against a `Venue`/`SeatLayout`, a batch job bulk-inserts one `Ticket` row per seat (`status='AVAILABLE'`, with per-seat/section `price`) for that `event_id`, and only then does the event flip to `ON_SALE`.

This matches the storage sizing already assumed elsewhere (§4: *"1M events × avg 5K seats = 5B rows"*). It's a non-contended, offline write — nothing like the 80 bookings/s hot path — so it doesn't affect the single-shard CAS design below. The provisioning flow itself (organizer/admin tooling, how/when that batch job runs) is **explicitly out of scope** (§1) and not designed here; it's called out only because the Reserve transaction's correctness depends on it having already run.

### Ticket Booking

Each seat-for-an-event is a **Ticket row** with a `status`. Booking is a **state transition on those rows**, executed as a **single-shard atomic conditional update** in Postgres. Because tickets are sharded by `event_id`, every seat a user selects for one event lives on the same shard → one local ACID transaction, no distributed coordination.

```
Ticket state machine

   ┌───────────┐  reserve (CAS)   ┌───────────┐  payment ok   ┌──────────┐
   │ AVAILABLE │─────────────────▶│ RESERVED  │──────────────▶│  BOOKED  │
   └───────────┘                  └─────┬─────┘               └────┬─────┘
         ▲      hold expired / user     │                          │ refund /
         └──────── releases ────────────┘                          │ cancel
         ▲                                                         │
         └─────────────────────────────────────────────────────────┘
```

### The core concurrency problem

Two users click seat G-7 within the same millisecond. A naive "read status, if AVAILABLE then write RESERVED" is a **check-then-act race** — both read AVAILABLE, both write, seat double-sold.

### Options considered

| Option | Mechanism | Pros | Cons |
|---|---|---|---|
| **A. Pessimistic lock** | `SELECT … FOR UPDATE` then `UPDATE` in a transaction | Simple mental model; correct | Lock held across round-trips; must order locks to avoid deadlocks |
| **B. Optimistic concurrency** | Read `version`; `UPDATE … WHERE version = :v`; retry on 0 rows | No lock wait; great under low contention | Under high contention most attempts fail and retry — wasted work |
| **C. Conditional atomic update (CAS)** | `UPDATE … WHERE status='AVAILABLE'`; check affected row count | One statement, one round-trip; no read-modify-write window; the DB is the arbiter | Need all-or-nothing handling for multi-seat requests |
| **D. Distributed lock (Redis `SET NX PX`)** | Lock `seat:{id}` before touching DB | Fast rejection; offloads DB | Second source of truth; Redis async replication can lose locks on failover — **cannot be the correctness mechanism alone** |
| **E. Serialize per event** (single-writer actor / Kafka partition per event) | All writes for an event processed sequentially | No locks at all; deterministic ordering | Adds latency/async UX; a single consumer becomes the throughput ceiling |

**Recommended:** **C** (with **A**'s lock ordering for multi-seat), DB as the sole source of truth. **D** may be layered on as a fast-path filter (Q4), never as the guarantee.

### The reserve transaction

```sql
BEGIN;

-- Lock rows in a deterministic order to prevent deadlocks between
-- users requesting overlapping seat sets. {eg: {G-7, G-8} , user2: {G-8, G-7}}
-- ticket_id here is the internal surrogate key (NOT the seat label like "G-7") — e.g. seats
-- G-7 and G-8 might map to ticket_ids {1042, 1043}; 
-- one user's request array might arrive as {1042,1043}, another's as {1043,1042}.
SELECT ticket_id FROM tickets
 WHERE event_id = :event_id AND ticket_id = ANY(:ticket_ids)
 ORDER BY ticket_id
 FOR UPDATE;

UPDATE tickets
   SET status          = 'RESERVED',
       booking_id      = :booking_id,
       hold_expires_at = now() + interval '10 minutes',
       version         = version + 1
 WHERE event_id   = :event_id
   AND ticket_id  = ANY(:ticket_ids)
   AND ( status = 'AVAILABLE'
         OR (status = 'RESERVED' AND hold_expires_at < now()) );   -- lazy expiry

-- Application checks: affected_rows == len(ticket_ids)?  else ROLLBACK → 409

INSERT INTO bookings (booking_id, user_id, event_id, status, total_amount,
                      idempotency_key, expires_at)
VALUES (:booking_id, :user_id, :event_id, 'PENDING',
        (SELECT sum(price) FROM tickets WHERE ticket_id = ANY(:ticket_ids)),
        :idempotency_key, now() + interval '10 minutes');

INSERT INTO outbox (aggregate_id, type, payload)
VALUES (:event_id, 'SeatsReserved', :payload);

COMMIT;
```

### Why this is correct (Staff-level detail)

- **All-or-nothing:** if 1 of 2 seats is taken, `affected_rows = 1 ≠ 2` → ROLLBACK releases the other. The couple never ends up split.
- **No lost updates under READ COMMITTED:** when two transactions target the same row, the second waits on the row lock; after the first commits, Postgres **re-evaluates the `WHERE` clause against the new row version** (EvalPlanQual). The second sees `status='RESERVED'` → 0 rows → clean 409. No SERIALIZABLE needed.
- **Single clock source:** `now()` comes from the shard's primary, not app servers — hold expiry comparisons can't be broken by app-server clock skew.
- **Lazy expiry in the predicate:** an abandoned hold is reclaimable the instant it expires, even if the cleanup job is down.
- **Defense in depth:** a `booked_tickets(ticket_id PRIMARY KEY, booking_id)` ledger row is inserted on confirmation — a unique-constraint violation makes a double-sell physically impossible even if application logic regresses.
- **Idempotency:** `bookings.idempotency_key UNIQUE` — a retried/double-clicked reserve returns the existing booking instead of creating a second hold.

### Where does Redis fit into Reserve?

Redis touches the Reserve path in two places, neither of which is the correctness mechanism:

1. **Admission check, before Reserve runs at all.** For a queue-enabled event, the request must carry a valid admission token (checked at the Gateway) before it ever reaches the Booking Service. That token comes from the waiting room's `Redis (queue)` — it's a traffic-shaping gate, not part of seat-locking.
2. **Optional fast-path lock on the seat (Option D above).** `SET seat:{id} {bookingId} NX PX ...` can reject obvious losers before they hit Postgres, saving a round-trip under extreme contention. It is explicitly **not** trusted for correctness — Redis's async replication can drop a lock on failover, so it's layered on as an optimization only.

The actual guarantee is the Postgres CAS `UPDATE ... WHERE status='AVAILABLE'` above. If Redis is down, stale, or loses a lock, that conditional update alone still prevents a double-sell — worst case, Reserve is slower under peak load, never incorrect.

### Guardrails

- Max 8 seats per booking; max 1 active hold per (user, event) — prevents **hold hoarding** (a bot reserving 5,000 seats to resell, letting them expire, re-reserving).
- Reserve requires a valid admission token when the event is queue-enabled.

### Q3 Follow-up: How do you make Reserve itself atomic for multi-seat selections?

> **Follow-up. How would you make the reserve call itself atomically succeed or fail for a multi-seat selection, so a user does not end up holding only a subset of the seats they clicked?**

**1. One set-based `UPDATE` over the whole seat set, not a per-seat loop.** The reservation isn't "lock seat 7, then lock seat 8, undo if the second fails." The `WHERE ticket_id = ANY(:ticket_ids)` clause matches the entire requested set in a single statement. A per-seat loop has a window between seat 1 committing and seat 2 being attempted where seat 1 is already held while seat 2's fate is unknown — exactly the "holding a subset" bug. A single set-based statement has no such window.

**2. The row-count check — not the `UPDATE` itself — is what enforces all-or-nothing.** This is the detail that's easy to miss: a multi-row `UPDATE` does **not** guarantee "all requested rows or none." If ticket_ids `{7,8}` are requested and seat 8 was taken a moment earlier, Postgres will happily update just row 7 — that's a perfectly valid execution of the statement, having touched 1 of 2 rows. The all-or-nothing guarantee comes from the application wrapping the `UPDATE` in a transaction and explicitly checking `affected_rows == len(ticket_ids)`:
   - Match → proceed to `INSERT INTO bookings` + `INSERT INTO outbox`, then `COMMIT`.
   - Mismatch → `ROLLBACK`, which undoes the write to seat 7 even though it was individually valid. The user ends up holding zero seats, not one.

**3. Deterministic lock ordering (`ORDER BY ticket_id FOR UPDATE`) prevents the race from corrupting that count.** Two overlapping multi-seat requests (e.g. `{7,8}` vs `{8,7}`) could lock their seats in opposite order and deadlock. Sorting ticket_ids before locking forces every transaction to acquire locks in the same global order, so concurrent requests queue linearly instead of cycling — avoiding spurious transaction kills under load that would otherwise surface as failures on exactly this multi-seat path.

**4. This is cheap only because of single-shard, single-node ACID — no 2PC or saga needed.** Because tickets are sharded by `event_id`, every seat a user could select for one event lives on the same Postgres shard. The multi-seat reserve is a local transaction; Postgres's native commit/rollback is sufficient. If seats could span shards, this would require a two-phase commit or a saga with compensating actions (reserve 7, reserve 8, and explicitly *un*-reserve 7 if 8 fails) — slower, more complex, and still exposed to getting stuck mid-compensation. Sharding by `event_id` is what avoids that entire class of machinery.

**5. The booking row and the outbox event ride in the same transaction as the ticket update.** So "seats held," "a booking record exists," and "the event announcing the hold will be published" are one atomic unit — never a booking with no held seats, or held seats with no booking, or a published `SeatsReserved` for a reservation that got rolled back (the transactional outbox pattern, §5 modeling choices).

**6. Idempotency is a separate concern, for client retries rather than partial seat sets.** `bookings.idempotency_key UNIQUE` protects against a double-click or timeout-retry creating a *second* booking for the same intent — it doesn't by itself prevent partial seat holds; that's items 1–5.

**Net result:** all seats the user clicked become `RESERVED` under one `booking_id` in a single commit, or none of them change state and the client gets one `409` — no code path leaves a user holding a strict subset of what they clicked.

---

## 12. Deep Dive Q4: Two-Phase Booking — Never Lose a Seat at Checkout

> **Q4. Implement a two-phase booking process: (1) Seat Reservation — temporarily hold selected seats; (2) Booking Confirmation — finalize purchase within a time limit. How would you design this to prevent users from losing seats during checkout?**

### The problem, precisely

A hold must be:
1. **Exclusive** — no one else can take the seat while held.
2. **Bounded** — abandoned holds must return seats to the pool (otherwise inventory leaks).
3. **Safe at the boundary** — a user who is *in the middle of paying* at 9:58 into a 10:00 hold must not lose the seat at 10:00 while the bank processes the card. *This is the real question.*

### Options for implementing the hold

| Option | Mechanism | Pros | Cons |
|---|---|---|---|
| **A. Long-running DB transaction / row lock** held for 10 min | `SELECT FOR UPDATE`, keep transaction open during checkout | Trivially exclusive | ❌ Holds connections for minutes → pool exhaustion; locks lost if app crashes or connection drops; untenable at scale |
| **B. Status + `expires_at` in DB, cron sweeper** | `RESERVED` + timestamp; cron resets expired rows every minute | Durable; DB is single source of truth | Seats stuck up to one sweep interval; sweeper is a correctness dependency; sweep scans are expensive |
| **C. Status + `expires_at` with lazy expiry** | Expired holds treated as `AVAILABLE` in the reserve predicate; sweeper only for hygiene/notifications | Durable, exact expiry, no correctness dependency on background jobs | Seat map needs a separate "released" signal for UX |
| **D. Redis distributed lock with TTL** | `SET seat:{id} {bookingId} NX PX 600000` | Auto-expiry; very fast; offloads DB | Two sources of truth (Redis + DB) must agree; Redis failover with async replication can lose locks → two users hold the "same" seat; TTL ≠ payment state |
| **E. Pay first, allocate after** | Charge card, then try to allocate seat | No holds at all | ❌ Terrible UX + refund storms during on-sales when allocation fails |

**Recommended:** **C as the source of truth**, optionally **D as a front filter** (reject obvious losers before they reach the DB during peak), plus an **explicit payment-pending state** to protect users at checkout.

### The booking state machine (the answer to "don't lose seats")

```
                 reserve (CAS)                checkout started
   (none) ─────────────────────▶ PENDING ───────────────────────▶ PAYMENT_PENDING
                                   │  hold: 10 min                  │  hold extended by
                                   │                                │  grace (e.g. +5 min)
                     expires_at    │                                │
                     passes        ▼                                │ webhook: succeeded
                                EXPIRED                             ▼
                              (seats freed)                     CONFIRMED  (seats BOOKED)
                                                                    │
                                   PAYMENT_PENDING + webhook: failed│ / grace exceeded
                                   ───────────────────────────────▶ EXPIRED (+ void/refund)
```

### Mechanisms that prevent losing a seat mid-checkout

**1. Visible, honest timer.** The reserve response includes `expiresAt` (server time). The client shows a countdown computed from server time, not the device clock. Users aren't surprised.

**2. Payment-pending grace extension (the critical one).** When the user clicks "Pay", `POST /checkout` atomically:

```sql
UPDATE bookings
   SET status = 'PAYMENT_PENDING',
       expires_at = now() + interval '5 minutes'           -- grace window
 WHERE booking_id = :id AND status = 'PENDING' AND expires_at > now();

UPDATE tickets
   SET hold_expires_at = now() + interval '5 minutes'
 WHERE booking_id = :id AND status = 'RESERVED';
```

This only succeeds if the hold is **still valid** — so we never start charging for a seat already lost. Once it succeeds, the bank's 3-D Secure flow, slow networks, and webhook delays are covered by the grace window. The extension is **one-shot** (only `PENDING → PAYMENT_PENDING`), so it can't be abused to hold seats indefinitely.

**3. Idempotent payment.** The PaymentIntent is created with `idempotency_key = booking_id`. Double-clicking "Pay" or a client retry produces one charge, not two.

**4. Webhook as the authority, idempotently applied.**

```sql
-- on payment_intent.succeeded (delivered at-least-once)
UPDATE tickets  SET status = 'BOOKED', hold_expires_at = NULL
 WHERE booking_id = :id AND status = 'RESERVED';
UPDATE bookings SET status = 'CONFIRMED'
 WHERE booking_id = :id AND status = 'PAYMENT_PENDING';
INSERT INTO booked_tickets (ticket_id, booking_id) ... ON CONFLICT DO NOTHING;
-- second delivery: 0 rows affected → no-op, return 200
```

**5. Reconciliation for lost webhooks.** A job scans `PAYMENT_PENDING` bookings near grace expiry and **queries the payment provider directly**. Paid → confirm. Unpaid → expire and void the authorization.

**6. Compensating action for the residual race.** If, despite everything, a payment succeeds for a booking whose seats were reclaimed (e.g., a webhook arrives hours late after a provider outage), the system **refunds automatically** and notifies the user — a business-level compensation (saga), never a double-sell. Prefer **authorize-then-capture**: authorize the card during checkout, **capture only after** the seats flip to BOOKED. A voided authorization is invisible to the user; a refund is not.

**7. Session resilience.** The `bookingId` is stored client-side (and in the user's account), so a page refresh or app crash **resumes** the same hold via `GET /bookings/{id}` instead of starting over. Navigating away doesn't release seats — only an explicit `DELETE` or expiry does.

**8. Fencing.** Every confirm operation is scoped by `booking_id` *and* expected state. A stale actor (old tab, duplicate webhook, retried worker) cannot overwrite a newer owner's state — the `booking_id` acts as a fencing token on the ticket row.

### Why not just Redis TTL locks? (expect this follow-up)

Redis TTL expiry is wall-clock-driven and **knows nothing about payment state**: a lock that expires while the bank is approving the card hands the seat to someone else. Redis replication is asynchronous, so a primary failover can drop recently acquired locks. Using Redis as a **pre-filter** (reject requests for seats with a live lock key before hitting Postgres) is a fine peak optimization; the **guarantee** must live in the DB where the seat state and the booking state change in the same transaction.

### Expiry hygiene (not correctness)

A delayed-job mechanism (Redis ZSET keyed by `expires_at`, or SQS delay queue) fires at hold expiry to flip `PENDING → EXPIRED`, emit `SeatsReleased` via outbox, and update the seat map. If it's late or down, **correctness is unaffected** — lazy expiry in the reserve predicate already treats the seats as free.

---

## 13. Deep Dive Q5: Scaling Reads to 10M Concurrent Users

> **Q5. How can your design scale to support up to 10M concurrent users reading event data? Focus on optimizing the database and read flow.**

### Load model

```
10M concurrent × 1 req / 10s           = 1,000,000 RPS at the edge
CDN hit ratio 90%                      →   100,000 RPS to origin
Redis hit ratio 95%                    →     5,000 RPS to DB replicas
```

The design goal: **the database should see almost none of the read traffic**, and what it sees should be cheap primary-key lookups on replicas.

### Layered read architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│ L0  CDN edge         event pages, layouts, images, popular searches  │ ~90%
├──────────────────────────────────────────────────────────────────────┤
│ L1  In-process cache  hot event docs, 1–5s TTL, per service pod      │ absorbs hot keys
├──────────────────────────────────────────────────────────────────────┤
│ L2  Redis Cluster     event:{id}, seatmap:{id}:bitmap                │ ~95% of origin
├──────────────────────────────────────────────────────────────────────┤
│ L3  Postgres replicas PK lookups on miss only                        │ ~0.5%
├──────────────────────────────────────────────────────────────────────┤
│     Postgres primary  writes only — never serves browse reads        │
└──────────────────────────────────────────────────────────────────────┘
```

### Techniques and their trade-offs

| Technique | What it solves | Trade-off |
|---|---|---|
| **CDN caching of event JSON** (`max-age=60, stale-while-revalidate`) | Bulk volume, global latency | Up to 60s staleness; explicit purge on edits |
| **Immutable, versioned seat layouts** | Biggest payload (MBs) served once per client, forever | Layout change → new URL (by design) |
| **Static/dynamic split** | Availability bitmap (~5KB) is the only volatile piece | Two requests instead of one |
| **Redis cache-aside + CDC invalidation** | Origin load; fresh after edits | Invalidation pipeline to operate |
| **Read replicas** | Cache-miss traffic off the primary | Replica lag (seconds) — acceptable for metadata |
| **Request coalescing (singleflight)** | **Cache stampede**: 50K simultaneous misses on `event:123` after expiry become 1 DB query | Small added latency for waiters |
| **Probabilistic early refresh / soft TTL** | Prevents synchronized expiry of hot keys | Slightly more refresh work |
| **L1 in-process cache** | **Hot key**: one event key at 500K RPS overloads a single Redis shard | Per-pod staleness (seconds) |
| **Hot-key replication** (`event:123#0..N`, random read suffix) | Spreads a single key across Redis shards | N× write cost on update |
| **Precomputed availability bitmap** | Never `SELECT` 70K ticket rows per viewer | Projection must be kept in sync (Q7) |
| **Connection pooling (PgBouncer)** | Thousands of pods × connections exhausting Postgres | Transaction-mode pooling limits session features |
| **Pre-warming & pre-scaling** before announced on-sales | Cold caches and autoscaler lag at T0 | Cost of idle capacity for a few hours |

### Database-specific optimizations

- **Event/venue data is tiny (~5 GB/yr)** — it fits entirely in Redis and in every replica's buffer cache. Don't shard it; replicate it.
- **Tickets are sharded by `event_id`**, but browse traffic never reads them directly — it reads the **availability projection** in Redis, built from the outbox stream.
- **Covering indexes** for the few DB reads that remain: `(event_id, status) INCLUDE (seat_id)` for snapshot rebuilds.
- **Archive past events** to cold storage so the hot ticket tables stay small and index-resident.

### The hot-event problem specifically

Sharding spreads *many* events; it does nothing for *one* huge event. The fixes are all about **replicating the hot data closer to readers** (CDN, L1, key replicas) and **limiting who can reach the dynamic parts** — which is exactly what the waiting room does (Q8): only admitted users fetch availability, capping that load at the admission rate.

---

## 14. Deep Dive Q6: Handling Complex Search Queries

> **Q6. How can you improve search to handle complex queries more efficiently? If your design already handles this, explain how.**

The HLD already uses Elasticsearch with CDC sync, which handles the core well. Here is how it handles **complex** queries efficiently, and what to add on top.

### 1. Denormalize at index time, not at query time

Elasticsearch has no efficient JOINs. The Indexer flattens event + venue + performer into one document. A query like *"jazz concerts at venues in Seattle with a performer named Kamasi"* becomes a single-document match. **Trade-off:** when a venue renames, every event at that venue must be re-indexed (the Indexer fans out via a `venue_id → event_ids` lookup).

### 2. Use the right query clause for each predicate

| Predicate | Clause | Why efficient |
|---|---|---|
| Free text | `multi_match` in `must`, field boosts (`performers^4`, `name^3`) | BM25 scoring only where relevance matters |
| Date range, status, category | `filter` | No scoring; results cached as bitsets and reused across queries |
| Location radius | `geo_distance` on `geo_point` (BKD tree) | Log-time spatial pruning |
| Typos | `fuzziness: AUTO` | Edit-distance matching via automaton |
| Price range | `range` filter on `min_price` | BKD numeric index |

### 3. Ranking beyond text relevance

`function_score` combining BM25 with **popularity** (sales velocity), **proximity** (gaussian decay on distance), and **recency** (closer dates rank higher). Popularity is updated from `booking-events` in Kafka via **partial document updates** — batched, not per-ticket.

### 4. Facets and filters UI

`aggregations` on `category`, `city`, `genre`, and price buckets return facet counts in the same query — no second round-trip.

### 5. Autocomplete / typeahead

Separate lightweight index using `completion` suggester (FST in memory) or `search_as_you_type` / edge n-grams on `name` and `performers`. Typeahead fires per keystroke — it must be **orders of magnitude cheaper** than full search, and is cached at the CDN by prefix.

### 6. Caching search results

Normalize the query (lowercase, sort params, round geo to ~1km grid, bucket dates) → cache key. Popular queries ("taylor swift", "concerts this weekend seattle") are cached in Redis/CDN for 30–60s. Long-tail queries go straight to ES. **Trade-off:** sold-out status may lag by up to the TTL — acceptable because booking validates truth.

### 7. Scale the cluster sensibly

- **Shards:** size for ~10–50GB per shard; this corpus is small, so a handful of primary shards with **many replicas** (read-heavy).
- **Index aliases + time-based indices** (e.g., by event month) so past events drop out of the hot index cheaply; queries for "upcoming" hit only current indices.
- **Routing** (e.g., by region) when queries are naturally region-scoped, so a query hits one shard instead of fanning out to all.
- **`search_after`** instead of deep `from/size` pagination.

### 8. Freshness and consistency

CDC → Kafka → Indexer gives seconds of lag. Use the **outbox/CDC LSN as the document version** (`version_type: external`) so out-of-order Kafka redelivery can't overwrite a newer document with an older one.

### Options for the sync pipeline

| Option | Pros | Cons |
|---|---|---|
| Dual-write from app (DB + ES) | Simple | ❌ Inconsistent on partial failure; no replay |
| Periodic batch reindex | Simple, consistent snapshots | Minutes–hours of staleness |
| **CDC (Debezium) → Kafka → Indexer** | Reliable, ordered per key, replayable, decoupled | Pipeline to operate | ✅ |

---

## 15. Deep Dive Q7: Real-Time Seat Map

> **Q7. How can you make the seat map on the event page automatically refresh to display the latest seat availability in real time?**

### Answer

Seat-state changes are captured **transactionally** (outbox), published to Kafka `seat-changes` (keyed by `event_id`), aggregated into **versioned, batched deltas** per event, and pushed to browsers over **Server-Sent Events (SSE)**. Clients load a **snapshot** first, then apply deltas; a version gap triggers a snapshot reload.

```
Booking Service ─tx─▶ tickets + outbox
                          │ (relay / CDC)
                          ▼
            Kafka [seat-changes]  key = event_id  (per-event ordering)
                          │
                          ▼
            Seat-Map Aggregator (per event partition)
              • applies change to Redis bitmap  seatmap:{eid}  (SETBIT)
              • bumps seatmap:{eid}:version
              • batches changes per 500ms–1s → one delta message
                          │
                          ▼  Redis Pub/Sub channel  seatmap:{eid}
                          │  (or internal broadcast)
          ┌───────────────┼────────────────┐
          ▼               ▼                ▼
     SSE node 1      SSE node 2   …   SSE node N    (~50K connections each)
          │               │                │
          ▼               ▼                ▼
        clients         clients          clients
```

### Transport options

| Option | Pros | Cons | Fit |
|---|---|---|---|
| **Short polling** (every 2–5s) | Trivial; stateless; CDN-cacheable for 1s | Wasteful; up to N seconds stale; 1M viewers = 200K–500K RPS | ✅ Fallback / degraded mode |
| **Long polling** | Fewer empty responses | Connection churn; awkward with load balancers | ⚠️ |
| **SSE** | Server→client is exactly the need; HTTP-native; auto-reconnect with `Last-Event-ID`; simple through proxies | One-directional (fine here); some per-browser connection limits on HTTP/1.1 (use HTTP/2) | ✅ **Chosen** |
| **WebSockets** | Bidirectional, low overhead | Bidirectionality unused; more complex infra (sticky sessions, custom protocols) | ⚠️ Overkill |

### Protocol details

```
1. Client: GET /availability            → { version: 4812, bitmap }
2. Client: GET /availability/stream     (Last-Event-ID: 4812)
3. Server pushes:
     id: 4813
     event: delta
     data: { "from": 4812, "to": 4813, "changes": [[1043,"R"],[1044,"R"],[2210,"A"]] }
4. Client: if delta.from != localVersion → refetch snapshot (gap detected)
5. On reconnect: browser sends Last-Event-ID; server replays from a short
   ring buffer of recent deltas, or tells the client to resnapshot.
```

### Scaling the fan-out (hot event)

- **Batching** turns ~150 changes/s into **1 message/s per event**, independent of sale rate.
- **Waiting room bounds the audience:** only admitted users (thousands, not millions) see the interactive seat map. Queued users see a static page. This is the single biggest lever.
- **Degraded granularity** under extreme load: push **section-level counts** ("Section 104: 12 left") instead of per-seat bits.
- **Horizontal SSE tier:** stateless besides connections; each node subscribes once per active event channel and fans out locally. Draining a node → clients reconnect elsewhere with `Last-Event-ID`.

### Correctness stance

The seat map is a **hint (AP/EL)**. A seat can be shown green and be taken 200ms later — the reserve call returns a clean 409 with the specific unavailable seats, and the UI highlights them and suggests nearby alternatives. **Never** let real-time infrastructure be part of the double-sell guarantee.

### Freeing seats on expiry

Lazy expiry doesn't produce an event. The expiry scheduler (Q4) emits `SeatsReleased` through the outbox at `expires_at` so seats turn green on the map promptly.

---

## 16. Deep Dive Q8: Virtual Waiting Room

> **Q8. How would you implement a virtual waiting room that queues users for popular events and grants access based on their queue position?**

### Goals

1. **Protect the backend** — admit users at the rate booking + inventory can sustain.
2. **Fairness** — deterministic, explainable order; bots don't win by being faster.
3. **Good UX** — show position and ETA; don't let users "lose their place" by refreshing.
4. **Stop early** — when inventory is gone, tell everyone in line immediately.

### Architecture

```
          T-30 min: event page shows "Join waiting room"
                         │
Client ──▶ CDN/Edge  (static waiting-room page; CAPTCHA/PoW;
                      edge stamps signed arrival token {uid, event, ts})
                         │
                         ▼
               Waiting Room Service ──▶ Redis (queue state per event)
                         ▲   ▲
       status polls/SSE  │   │ admit N users / interval
                         │   │
               Admission Controller ◄── metrics: booking p99, DB load,
                         │               active holds, remaining inventory
                         ▼
          issues signed Admission Token (JWT: uid, event, device, exp=10–15 min)
                         │
                         ▼
          API Gateway validates token on /availability and /bookings/reserve
                         │
                         ▼
               Seat map + Booking Service (protected)
```

### Assigning queue order — options

| Option | Mechanism | Pros | Cons |
|---|---|---|---|
| **A. Redis ZSET by join timestamp** | `ZADD q:{eid} ts uid`; position = `ZRANK` | Precise FIFO; easy removal | Join burst hammers one key/shard; O(log N) per op; clock source must be uniform |
| **B. Monotonic counter** | `INCR q:{eid}:seq` → my number; position = my_number − admitted_cursor | O(1); trivial position math; tiny memory | Single hot key at join burst (~100K ops/s per Redis shard); abandoned entries handled lazily (skipped at admission) |
| **C. Edge-stamped signed arrival tokens** | CDN edge signs `{uid, event, arrival_ts}`; no central write on join | Join scales with the CDN, not Redis; tamper-proof; survives refresh | Ordering only as good as edge clock sync (ms-level — fine); admission needs to scan/bucket by timestamp |
| **D. Randomized lottery for pre-on-sale arrivals** | Everyone who joins before T0 gets a random position at T0; after T0, FIFO | Removes incentive to start 10,000 bots early or hammer refresh at T0; perceived fairness | Users who arrived "first" may feel cheated — product must communicate it |

**Recommended:** **D + C/B hybrid**. Pre-queue (joined before T0) → random shuffle into numbered positions at T0. Post-T0 joiners → appended FIFO by edge-stamped arrival time. Internally, positions are numbers; **position for display = my_number − admitted_cursor** (lazily computed — no need to update millions of entries every time the line moves).

### Admission control

```
Every second, Admission Controller:
  capacity = min(
      target_active_shoppers − currently_active,        # concurrency cap
      booking_service_headroom(p99 latency, error rate),
      remaining_inventory × overbooking_factor          # e.g. 1.5× unsold seats
  )
  admitted_cursor += max(capacity, 0)
  publish admitted_cursor  → Waiting Room Service / clients
```

- **Leaky-bucket admission** decoupled from user actions — feedback-driven, not a fixed rate.
- **Inventory-aware**: admitting 2M users for 70K seats is pointless. Admit roughly as many as can plausibly buy; when `remaining_inventory == 0` and no active holds → broadcast **SOLD_OUT** to the whole queue (cheap: one flag, read by every status poll).
- If holds expire and seats return, the controller admits more.

### Admission token

```json
{ "sub": "user_42", "evt": "e_123", "dev": "fp_9a1c", "iat": 1793..., "exp": 1793...+900,
  "jti": "adm_7f3e" }
```

- **Stateless validation** at the gateway (signature + `exp` + `evt` match + `sub` match the auth JWT) — no Redis lookup on the hot path.
- **Bound to user and device** so tokens can't be sold or shared.
- **Expires** (10–15 min): an admitted user who wanders off frees capacity; the controller counts active shoppers by issued-and-unexpired tokens.
- **Single active token per user per event** (`jti` tracked in Redis with TTL) to stop one account opening 50 tabs.

### Position updates to the client

- Clients poll `/queue/{eid}/status` every 10–30s with jitter (cacheable per position bucket), or subscribe via SSE to a **broadcast of `admitted_cursor`** — every client computes its own position locally. **One broadcast message updates millions of clients** instead of millions of personalized messages.
- Refreshing the page doesn't lose the spot: the signed queue ticket in a cookie/local storage identifies the position.

### Bot and abuse resistance

- CAPTCHA / proof-of-work at join (raises cost per bot entry).
- **Verified Fan** pre-registration: only pre-approved accounts get a join link/code for the hottest events.
- One queue entry per account; device fingerprinting; per-IP/ASN limits at the edge.
- Post-admission behavior analysis (inhuman click timing → token revoked).

### Failure handling

| Failure | Behavior |
|---|---|
| Waiting Room Redis primary fails | Replica promotion; queue numbers are also encoded in signed client tokens, so state can be **reconstructed** — users don't lose positions |
| Admission Controller down | Admission pauses (fail closed); queue page shows "hold tight" — prefer pausing over a flood that breaks booking |
| Booking service degraded | Controller sees p99 rise → admits fewer (backpressure) |
| CDN edge outage | Other PoPs serve the static page; edge tokens are verifiable anywhere with the shared key |

### When to enable

Per-event flag `queue_enabled`, set by ops for announced high-demand on-sales, **plus** automatic activation when concurrent viewers of an event exceed a threshold (e.g., 5× seat count).

---

## 17. CAP / PACELC Positioning

| Component | CAP | PACELC | Reasoning |
|---|---|---|---|
| **Ticket inventory / reserve / confirm** (Postgres shard per event) | **CP** | PC/EC | Double-sell is catastrophic; refuse writes during partition. Sync standby for RPO=0 on failover. |
| **Bookings & payment state** | **CP** | PC/EC | Money and ownership must agree; reconciliation with the provider for edge cases. |
| **Waiting room order & admission cursor** | **CP-leaning** | PC/EC | Fairness promise; on failure pause admission rather than reorder or flood. |
| Admission token validation (gateway) | AP | EL | Stateless signature check — always available, always fast. |
| Event metadata (CDN + Redis + replicas) | **AP** | PA/EL | Stale description is harmless; availability matters. |
| Search (Elasticsearch via CDC) | **AP** | PA/EL | Seconds of index lag acceptable; booking validates truth. |
| Seat-map projection & SSE | **AP** | PA/EL | Hint only; a stale green seat gets a clean 409. |
| Notifications | AP | PA/EL | At-least-once with idempotent consumers; delays acceptable. |

**Shape of the system:** AP/EL everywhere *except* three CP islands — inventory, payments/bookings, and queue ordering. Naming the islands explicitly is the Staff signal.

---

## 18. Failure Modes & Mitigations

| Failure | Symptom | Detection | Mitigation |
|---|---|---|---|
| **Double-click / client retry on reserve** | Two holds for one user | Duplicate idempotency keys in logs | `Idempotency-Key` UNIQUE on bookings; return existing booking |
| **Payment succeeds after hold expiry** | User charged, seat gone | Confirm webhook finds 0 RESERVED rows | `PAYMENT_PENDING` grace extension; authorize-then-capture; auto-void/refund compensation |
| **Webhook lost or duplicated** | Booking stuck `PAYMENT_PENDING` / double-processed | Age of PAYMENT_PENDING bookings | Idempotent state transitions; reconciliation job polling the provider |
| **Deadlock on overlapping seat sets** | Reserve errors under load | Postgres deadlock counter | Lock rows in sorted `ticket_id` order; retry once |
| **Hot event shard** | One Postgres primary at 100% | Per-shard CPU / lock waits | Waiting room caps writes; Redis pre-filter on held seats; dedicated shard for mega events |
| **Cache stampede on event page at T0** | DB spike when hot key expires | Miss rate spike | Request coalescing; soft TTL / early refresh; pre-warm before on-sale |
| **Hot key in Redis** | One Redis shard saturated | Per-shard ops/s | L1 in-process cache; key replication with random suffix |
| **Dual-write inconsistency** (DB updated, event not published) | Seat map / search diverge | Projection vs DB audits | Transactional outbox; CDC |
| **Outbox relay lag** | Seat map seconds behind | Relay lag metric | Scale relay; map is a hint; 409s still correct |
| **SSE node crash** | Clients disconnected | Connection drop rate | Auto-reconnect with `Last-Event-ID`; ring-buffer replay or resnapshot |
| **ES index lag** | Sold-out events still shown | CDC lag metric | Booking validates; sold-out status surfaced on event page |
| **Hold hoarding by bots** | Inventory frozen, then dumped | Holds per account / expiry rate | Max seats per booking; 1 active hold per user/event; admission tokens; abuse scoring |
| **Clock skew across app servers** | Holds expire early/late | Expiry vs timestamp audits | All expiry math via `now()` on the shard primary (single clock); client timers use server `expiresAt` |
| **Postgres primary failover** | Brief reserve errors | Health checks | Sync standby (no committed hold lost); clients retry with same idempotency key |
| **Waiting-room Redis loss** | Queue positions lost | Replica lag / failover alert | Positions encoded in signed client tokens → reconstructable; pause admission during recovery |
| **Admission token sharing/resale** | Scalpers buying with others' tokens | Token used from multiple devices | Bind token to user + device fingerprint; single active `jti` |
| **Payment provider outage** | Checkouts failing | Provider error rate | Circuit breaker; extend grace for affected bookings; secondary provider |

**Pattern class (Staff framing):** *hold hoarding*, *payment-after-expiry*, *T0 thundering herd*, and *hot event key* are this domain's equivalents of the crawler's spider trap and the news feed's celebrity fan-out — adversarial or pathological load concentrated on one key. Naming them as a class shows you design for the pathological case, not the average.

---

## 19. Scalability & Sharding

### Sharding strategy

| Data | Shard key | Rationale | Consequence |
|---|---|---|---|
| Tickets, bookings, outbox | `event_id` | All seats of an event co-located → single-shard ACID reserve | A mega event is a hot shard → mitigated by waiting room, not resharding |
| "My bookings" by user | Secondary index / CQRS read model keyed by `user_id` (fed from `booking-events`) | Avoids scatter-gather across all event shards | Eventually consistent order history (seconds) |
| Events / venues | Not sharded — replicated | Tiny dataset | — |
| Kafka `seat-changes` | `event_id` | Per-event ordering of deltas | Hot partition for a mega event — 150 msgs/s is still trivial |
| Elasticsearch | Default hash + replicas; optional region routing | Read-heavy | — |
| Waiting room Redis | `event_id` | Each queue independent | One mega queue per shard — fine at ~1.4GB |

**The sharding question that matters:** sharding by `event_id` makes "reserve seats" cheap and "list a user's bookings" a fan-out — so you build a user-keyed read model. Choosing what becomes expensive is the Staff decision.

### Horizontal scaling plan

| Component | Strategy |
|---|---|
| Event / Search / Booking services | Stateless; autoscale; **pre-scale on on-sale schedule** |
| CDN | Managed; pre-warm event pages and layouts |
| Redis cache | Cluster mode; add shards; L1 caches for hot keys |
| Ticket DB | Add shards; consistent mapping `event_id → shard` in a directory service; new events placed on least-loaded shard |
| SSE tier | Add nodes; connection-count-based LB |
| Elasticsearch | Add replicas for QPS; reindex via alias swap |

### Backpressure chain

```
DB lock waits ↑ → Booking p99 ↑ → Admission Controller admits fewer
→ queue grows (cheap, at the edge) → backend stays healthy
```

Queueing at the **cheapest layer** (CDN-served waiting page) instead of the **most expensive** (DB lock waits) is the core scalability idea.

---

## 20. Senior vs Staff Answer Differentiators

### Senior-level expectations

- Identify services: Event, Search, Booking; Postgres + Redis + Elasticsearch
- Use a status column and a transaction to prevent double booking
- Use a TTL (Redis lock or `expires_at`) for temporary holds
- Cache event data; use Elasticsearch for search
- Mention WebSockets/SSE for live seat map; mention a queue for popular events

### Staff-level expectations (additional)

| Topic | What Staff adds |
|---|---|
| Framing | Names the anchor constraint (AP read path vs CP inventory island) up front and derives decisions from it |
| Reserve correctness | Conditional UPDATE + rowcount all-or-nothing; EvalPlanQual under READ COMMITTED; sorted lock order; unique-constraint ledger as last line of defense |
| Holds | Lazy expiry in the predicate (no correctness dependency on sweepers); single clock source |
| Checkout | `PAYMENT_PENDING` grace extension; authorize-then-capture; idempotent webhooks; reconciliation; refund compensation |
| Redis locks | Explains *why* Redis can't be the guarantee (async replication, TTL blind to payment state); uses it only as a pre-filter |
| Dual writes | Transactional outbox / CDC for every derived view |
| Read scale | Distinguishes horizontal scale from **hot key** scale; L1 + key replication + coalescing; static/dynamic split; immutable layouts |
| Seat map | Snapshot + versioned deltas + gap detection; batching; audience bounded by waiting room; map is a hint |
| Waiting room | Inventory-aware, feedback-driven admission; lottery for pre-queue; edge-stamped tokens to avoid a hot counter; lazy position math; signed, device-bound tokens |
| Capacity | Shows that average writes are trivial and the problem is concentration |
| CAP | Per-component table with exactly three named CP islands |
| Operations | Pre-scaling on known on-sale schedules; per-event dashboards; double-sell canary |

### The single most important Staff differentiator

**Recognizing that the waiting room is the scaling solution for the write path — not a UX feature.** It converts an impossible contention problem (2M users hammering one shard) into a small, controlled one (a few thousand admitted shoppers), and it bounds the real-time fan-out and the availability reads at the same time. Senior candidates add a queue because "Ticketmaster has one"; Staff candidates derive it from the capacity math.

---

## 21. Interview Time Allocation

For a 45-minute session:

| Phase | Time | Focus |
|---|---|---|
| Requirements & scope | 4 min | 4 core FRs, CP inventory vs AP reads, out-of-scope list |
| Capacity estimation | 3 min | 1M RPS reads, trivial average writes, concentration on one event, seat-map payload |
| Core entities + API | 5 min | Ticket per seat with status/expiry; two-phase booking endpoints; idempotency key |
| High-level design | 7 min | Boxes and arrows; walk each FR through it |
| Q3 + Q4 booking deep dive | 10 min | Conditional UPDATE, lazy expiry, PAYMENT_PENDING grace, webhook idempotency |
| Q5 read scaling | 4 min | CDN/L1/Redis layering, hot key, stampede |
| Q8 waiting room | 5 min | Ordering, admission control, tokens, bots |
| Q7 seat map / Q6 search | 4 min | SSE + deltas; ES + CDC — brief |
| Failure modes & CAP wrap-up | 3 min | Name the CP islands; 3–4 pathological cases |

**What to cut if short on time:** Q6 search internals and SSE protocol details. **Never cut:** the reserve transaction, the checkout grace mechanism, and the waiting room as write-path protection.

---

## 22. Quick-Reference Cheatsheet

```
KEY MECHANISMS
──────────────
Reserve:        UPDATE tickets … WHERE status='AVAILABLE' OR hold expired; rowcount == N else ROLLBACK
Lock order:     SELECT … ORDER BY ticket_id FOR UPDATE   (no deadlocks)
Expiry:         Lazy — expired hold == available in predicate; scheduler only for UX events
Checkout:       PENDING → PAYMENT_PENDING (+grace, one-shot) → CONFIRMED via signed webhook
Payment:        Idempotency key = booking_id; authorize → capture after BOOKED
Safety net:     booked_tickets(ticket_id PK) unique constraint
Derived views:  Transactional outbox / CDC → Kafka (key = event_id)
Seat map:       Snapshot (2-bit bitmap) + versioned SSE deltas, batched 1/s
Waiting room:   Lottery pre-T0 + FIFO after; position = my_number − admitted_cursor
Admission:      Feedback + inventory-aware leaky bucket; signed, device-bound JWT

KEY NUMBERS (illustrative)
──────────────────────────
10M concurrent → ~1M RPS edge → ~100K origin → ~5K DB
Avg bookings ~16 tickets/s; mega on-sale ~80 confirms/s
Seat map: 4.2MB JSON → 5KB gzipped bitmap + immutable CDN layout
Fan-out: 1M viewers × 1 delta/s ≈ 1M msgs/s; 50K conns/SSE node
Queue: 14M entries ≈ 1.4GB Redis
Hold: 10 min + 5 min payment grace; max 8 seats; 1 hold per user/event

KEY SYSTEMS
───────────
Inventory/bookings:  Postgres sharded by event_id (sync standby)
Event metadata:      Postgres (replicated) + Redis + CDN
Search:              Elasticsearch ← Debezium CDC ← Postgres
Events bus:          Kafka (seat-changes, booking-events, event-changes)
Real-time:           SSE tier + Redis Pub/Sub
Waiting room:        CDN edge + Redis + Admission Controller
Payments:            Stripe/Adyen (PCI offloaded), webhooks + reconciliation

CAP DECISIONS
─────────────
CP:  Ticket inventory · Bookings/payments · Queue ordering/admission cursor
AP:  Event metadata · Search · Seat map · Token validation · Notifications

STAFF SIGNALS TO HIT
────────────────────
✓ Anchor constraint: AP read path + CP inventory island
✓ All-or-nothing multi-seat CAS; EvalPlanQual; sorted locks
✓ Lazy expiry; single DB clock
✓ Payment-after-expiry race → PAYMENT_PENDING grace + auth/capture + compensation
✓ Why Redis locks aren't the guarantee
✓ Outbox for every derived view
✓ Hot key ≠ horizontal scale; L1 + key replication + coalescing
✓ Waiting room derived from capacity math as write-path protection
✓ Pathological-load pattern class: hoarding, T0 herd, hot event, late webhook
```

---

## 23. Related Guides

- **Web Crawler** — format template; per-domain politeness queues are the same "isolate the hot key" idea as per-event admission.
- **Distributed Unique ID Generation** — `booking_id`, `ticket_id`, and idempotency keys.
- **Apache Kafka** — `acks` + `min.insync.replicas` durability for `booking-events`; per-key ordering for `seat-changes`.
- **Sharding Strategies** — `event_id` sharding and the user-history fan-out it creates.
- **CAP / PACELC Theorem** — the three CP islands vs AP/EL read path.
- **Circuit Breaker Pattern** — payment-provider outages and failover.
- **Leaderboard System** — Redis sorted sets (queue ordering alternative).
- **Messaging Application (WhatsApp/MS Teams)** — connection-tier fan-out (SSE/WebSocket scaling).
- **News Feed / Twitter Fan-out** — celebrity hot-key fan-out ≈ mega-event seat map fan-out.
- **Notification System** — booking confirmations and e-ticket delivery.
- **Yelp (local business search)** — Elasticsearch geo queries and CDC indexing.

---

*Guide compiled for Senior & Staff-level system design interview preparation.*
*Topics: Ticketmaster, Seat Inventory, Two-Phase Booking, Distributed Concurrency, Waiting Room, Real-Time Fan-out, CAP/PACELC.*
