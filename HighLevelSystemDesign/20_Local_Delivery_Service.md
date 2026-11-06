# Local Delivery Service (Gopuff-style) — System Design Interview Guide
### Senior & Staff Engineer Level

> **Problem in one line:** Let a customer see which convenience-store items can reach their door within ~1 hour from one of 500+ micro distribution centers (DCs), and let them order those items with a guarantee that we never sell stock that doesn't exist.

> **Anchor constraint (drives every decision below):** The **read path** (browse availability) is huge, latency-sensitive, and tolerant of a few seconds of staleness. The **write path** (place order) is small but must be **strongly consistent** — overselling a physical item is a real customer-facing failure. Split them hard: **AP reads, CP writes.**

---

## Table of Contents

1. [Functional Requirements](#1-functional-requirements)
2. [Non-Functional Requirements](#2-non-functional-requirements)
3. [Capacity Estimation](#3-capacity-estimation)
4. [Core Entities](#4-core-entities)
5. [API / System Interface](#5-api--system-interface)
6. [Data Flow](#6-data-flow)
7. [High-Level Design](#7-high-level-design)
8. [Deep Dive Q1: Availability Within a Fixed Distance (30 miles)](#8-deep-dive-q1-availability-within-a-fixed-distance-30-miles)
9. [Deep Dive Q2: Only Items Deliverable Within 1 Hour](#9-deep-dive-q2-only-items-deliverable-within-1-hour)
10. [Deep Dive Q3: Placing Orders](#10-deep-dive-q3-placing-orders)
11. [Deep Dive Q4: Preventing Oversell & Race Conditions](#11-deep-dive-q4-preventing-oversell--race-conditions)
12. [Deep Dive Q5: Fast & Highly Available Availability Lookups](#12-deep-dive-q5-fast--highly-available-availability-lookups)
13. [NFR Deep Dive 1: Strong Consistency for Inventory (No Oversell)](#13-nfr-deep-dive-1-strong-consistency-for-inventory-no-oversell)
14. [NFR Deep Dive 2: Low-Latency, Highly Available Reads](#14-nfr-deep-dive-2-low-latency-highly-available-reads)
15. [NFR Deep Dive 3: Scalability Under Demand Spikes](#15-nfr-deep-dive-3-scalability-under-demand-spikes)
16. [CAP / PACELC Positioning](#16-cap--pacelc-positioning)
17. [Failure Modes & Mitigations](#17-failure-modes--mitigations)
18. [Scalability & Sharding](#18-scalability--sharding)
19. [Senior vs Staff Differentiators](#19-senior-vs-staff-differentiators)
20. [Interview Time Allocation](#20-interview-time-allocation)
21. [Quick-Reference Cheatsheet](#21-quick-reference-cheatsheet)
22. [Related Guides](#22-related-guides)

---

## 1. Functional Requirements

Four in scope (picked for the interview), the rest explicitly parked.

| # | Requirement | Why it's in scope |
|---|---|---|
| **FR1** | **Browse/search item availability** for a delivery location — only items that are in stock at a DC that can deliver to that address (≤ 30 mi, extended to ≤ 1 hr ETA) | The core read path; drives the geo + latency problems |
| **FR2** | **Place an order** for multiple items in one checkout; the order either fully succeeds or fully fails | The core write path; drives the consistency problem |
| **FR3** | **DC inventory management** — DC staff/WMS record restocks, picks, damages/shrinkage; changes flow to what customers see | Source of truth for stock; feeds the read path |
| **FR4** | **Order fulfillment & tracking** — order moves through a state machine (reserved → confirmed → picking → out for delivery → delivered/cancelled), driver assigned, customer sees live status | Closes the loop; releases/consumes reservations |

**Out of scope (say so out loud):** promotions/coupons, recommendations, subscriptions (Gopuff FAM), driver payroll, supplier procurement/replenishment forecasting, ratings & reviews, returns.

**Clarifying questions to ask the interviewer:**
- Is an order always fulfilled from **one DC**, or may it be split? → *Assume one DC per order (one driver, one bag). Discuss splitting as an extension.*
- Do customers see **exact quantities** or just in/out of stock? → *In/out plus "only N left" when low.*
- Is payment authorized at checkout and captured at delivery? → *Yes — auth at checkout, capture on delivery.*

---

## 2. Non-Functional Requirements

| # | Requirement | Target (illustrative) | Deep-dived? |
|---|---|---|---|
| **NFR1** | **Strong consistency for inventory** — never accept an order for stock that isn't there; no double-sell under concurrency | 0 oversells caused by software (physical shrinkage handled separately) | ✅ §13 |
| **NFR2** | **Low-latency, highly available availability reads** | p99 < 100 ms server-side; 99.99% availability for browse | ✅ §14 |
| **NFR3** | **Scalability under spikes** — evening peaks, Super Bowl, snowstorms (10×) | Browse ~12K QPS peak; orders ~120/s peak with headroom to 10× | ✅ §15 |
| NFR4 | Order placement latency | p99 < 1 s end-to-end incl. payment auth | Covered in §10 |
| NFR5 | Durability of orders/payments | No acknowledged order is ever lost (RPO = 0 for orders) | Covered in §13 |

**The tension to name up front:** NFR1 (CP) and NFR2 (AP) look contradictory. They're reconciled by **separating the read model from the write model** (CQRS): browse reads a fast, eventually-consistent projection; the order path does an authoritative, transactional check against the source of truth.

---

## 3. Capacity Estimation

> All numbers are **illustrative** (decimal/SI units). The point is the *shape*, not the exact figure.

```
USERS & TRAFFIC
  DAU:                         10M
  Availability queries/user:   ~10 (home, category, search, cart refresh)
  Availability queries/day:    100M
    avg QPS:                   100M / 86,400 ≈ 1,160 QPS
    peak (5× evening):         ≈ 6,000 QPS
    spike (10×, Super Bowl):   ≈ 12,000 QPS

ORDERS
  Orders/day:                  1M
    avg:                       ≈ 12 orders/s
    peak (10×):                ≈ 120 orders/s
  Line items/order:            ~5  → peak ≈ 600 inventory decrements/s (cluster-wide)
  Per DC (500 DCs):            1M / 500 = 2,000 orders/day/DC → peak ≈ 0.25–1 orders/s per DC
  Read:write ratio:            ~100:1

INVENTORY
  DCs:                         500
  SKUs per DC:                 ~10,000 (convenience assortment)
  Inventory rows:              500 × 10K = 5M rows × ~100 B ≈ 500 MB  → fits in RAM
  Availability BITMAP per DC:  10K SKUs × 1 bit = 1.25 KB
  Bitmap for ALL DCs:          500 × 1.25 KB ≈ 625 KB   ← fits in every API node's memory
  With 2-bit qty buckets:      (out / low / medium / plenty) ≈ 1.25 MB total

GEO / ETA PRECOMPUTE
  H3 resolution 8 cell:        ~0.74 km²
  Area of a 30-mile radius:    π × 48 km² ≈ 7,240 km² → ~10K cells per DC
  Total cells (with overlap):  ~2–3M cells
  cell → [(dc_id, drive_secs)] ≈ 3M × 3 DCs × 8 B ≈ 72 MB  → fits in Redis / memory

ORDER STORAGE
  ~2 KB/order (header + lines + status history)
  1M/day × 2 KB = 2 GB/day ≈ 730 GB/year → Postgres, partitioned by month, archived

DRIVER LOCATION (FR4)
  20K active drivers at peak × 1 ping / 5 s = 4,000 writes/s → Redis GEO + Kafka
```

**Key insight (Staff-level):** This is **not** a big-data problem. The entire inventory fits in memory and the entire availability state fits in ~1 MB. The hard problems are **(1) correctness under concurrency** on the write path and **(2) turning "deliverable in 1 hour" into a fast lookup**. Don't spend interview minutes on storage sharding math that doesn't matter here.

---

## 4. Core Entities

```
Item                 DistributionCenter         Inventory (source of truth)
──────────           ──────────────────         ──────────────────────────
item_id (PK)         dc_id (PK)                 dc_id      ┐ PK
name, brand          name                       item_id    ┘
category             lat, lng                   on_hand          INT
price_cents          address                    reserved         INT
image_url            status (OPEN/BUSY/CLOSED)  safety_stock     INT
attrs (age-gated?)   hours, timezone            version          BIGINT
                     prep_time_p50_secs (live)  updated_at
                                                CHECK (reserved <= on_hand)
                                                available := on_hand - reserved - safety_stock

Order                        OrderLine               Delivery / Driver
──────────────────           ─────────────           ─────────────────────
order_id (PK, Snowflake)     order_id                delivery_id
customer_id                  item_id                 order_id
dc_id                        qty                     driver_id
status (state machine)       unit_price_cents        status
idempotency_key (UNIQUE)                             last_lat, last_lng, eta
delivery_lat/lng, h3_cell
total_cents
payment_intent_id
reservation_expires_at
created_at, updated_at

Customer              InventoryEvent (append-only ledger)     CellReachability (precomputed)
───────────           ───────────────────────────────────     ──────────────────────────────
customer_id           event_id, dc_id, item_id                h3_cell (PK)
addresses[]           delta, reason (RESTOCK/PICK/RESERVE/    [(dc_id, drive_secs)] sorted
payment_methods[]       RELEASE/SHRINK/ADJUST), ref_id, ts    computed_at, traffic_profile
```

**Design notes:**
- **`on_hand` vs `reserved`** are separate columns. A reservation doesn't change physical stock; picking does. This makes cancellations and expiry trivial (release `reserved`) and keeps the physical count auditable.
- **`safety_stock`** absorbs physical-vs-system discrepancy (mis-shelved, damaged). Selling the last physical unit of a fast-moving SKU is how you get "item unavailable" calls from the picker.
- **`InventoryEvent`** ledger makes every change auditable and replayable — the inventory table is effectively a materialized balance of the ledger.

---

## 5. API / System Interface

### Customer-facing

```http
# FR1 — availability (location-scoped; no auth required to browse)
GET /v1/availability?lat=47.61&lng=-122.03
                    &max_eta_min=60          # or &radius_mi=30
                    &q=ice+cream&category=frozen
                    &cursor=abc&limit=50
→ 200 {
    "serving_dc_ids": ["dc_118", "dc_121"],
    "items": [
      { "item_id": "sku_9912", "name": "...", "price_cents": 599,
        "stock_level": "LOW",           # IN_STOCK | LOW | (out-of-stock items omitted)
        "eta_min": 28 }
    ],
    "next_cursor": "def",
    "as_of": "2026-11-01T20:58:03Z"     # staleness made explicit
  }

# FR2 — place order (idempotent)
POST /v1/orders
Idempotency-Key: 7c1e9a2f-...           # client-generated per checkout attempt
{
  "items": [ { "item_id": "sku_9912", "qty": 2 }, { "item_id": "sku_0042", "qty": 1 } ],
  "delivery_address_id": "addr_55",
  "payment_method_id": "pm_77",
  "expected_total_cents": 1797          # guards against price changes mid-checkout
}
→ 201 { "order_id": "ord_...", "status": "CONFIRMED", "dc_id": "dc_118", "eta_min": 31 }
→ 409 { "error": "INSUFFICIENT_STOCK",
        "unavailable": [ { "item_id": "sku_0042", "available": 0 } ],
        "suggestions": [ ... ] }
→ 402 { "error": "PAYMENT_DECLINED" }

# FR4 — tracking
GET  /v1/orders/{order_id}
GET  /v1/orders/{order_id}/stream       # SSE/WebSocket: status + driver location
POST /v1/orders/{order_id}/cancel       # allowed only before PICKING
```

### Internal / operator-facing

```http
# FR3 — DC inventory (WMS / handheld scanners)
POST /internal/v1/dcs/{dc_id}/inventory/events
{ "events": [ { "item_id": "sku_9912", "delta": +48, "reason": "RESTOCK", "ref": "po_123" },
              { "item_id": "sku_0042", "delta": -1,  "reason": "SHRINK",  "ref": "damage" } ] }

PATCH /internal/v1/dcs/{dc_id}/status   { "status": "BUSY", "prep_time_p50_secs": 900 }

# Driver app
POST /internal/v1/drivers/{driver_id}/location   { "lat": .., "lng": .., "ts": .. }
POST /internal/v1/deliveries/{id}/status          { "status": "PICKED_UP" }
```

**Interface decisions worth defending:**
- **`lat/lng` in the availability request, not `customer_id`** → the response is a pure function of (cell, filters), which makes it **cacheable and shareable across users** in the same H3 cell.
- **`Idempotency-Key` on POST /orders** → double-taps and client retries can't create duplicate orders or double reservations.
- **`expected_total_cents`** → if price/availability shifted between cart and submit, fail explicitly instead of silently charging a different amount.
- **`as_of` in availability responses** → makes eventual consistency explicit to the client.

---

## 6. Data Flow

### Flow A — Browse availability (FR1, read path)

```
1. Client sends GET /availability (lat, lng, filters)
2. API → snap (lat, lng) to H3 res-8 cell
3. Look up CellReachability[cell] → candidate DCs with drive time
4. Add live prep_time[dc] → keep DCs with total ETA ≤ 60 min (and status ≠ CLOSED)
5. OR-merge those DCs' availability bitmaps (in-memory snapshot)
6. Catalog search (q/category) → ranked candidate item_ids → filter by merged bitmap
7. Hydrate with catalog data (name, price, image) from catalog cache
8. Return page + as_of timestamp
```

#### Steps 2–4, in detail

**Step 2 — snap `(lat, lng)` to an H3 res-8 cell.** The customer's exact coordinates are continuous and effectively unique per request — two neighbors a few meters apart would otherwise never share a cache key or a lookup row. "Snapping" means running them through H3's `lat_lng_to_cell(lat, lng, resolution=8)` function, which deterministically maps any point inside a given hexagon to the same 64-bit cell ID (§3's capacity math uses res-8, ~0.74 km² per hexagon — small enough that "which DC can reach me" barely varies within one cell, but large enough that the whole world only needs ~2–3M cells, see §3 `GEO / ETA PRECOMPUTE`). This is a pure, local computation — no network call, no database lookup — it just turns "40.7128° N, 74.0060° W" into one fixed integer that two nearby customers will produce identically. That integer is the key used by the very next step.

**Step 3 — look up `CellReachability[cell]` → candidate DCs with drive time.** `CellReachability` is the precomputed entity defined in §4 (`Core Entities`): a table keyed by `h3_cell`, where each row holds `[(dc_id, drive_secs), ...]` sorted by drive time — i.e., "from this cell, DC #4 is 18 minutes away by road, DC #11 is 31 minutes away," etc. This is a plain **key lookup** (Redis, replicated into every API node's memory — §3 sizes the whole table at ~72 MB), not a geometry computation at request time. Where the numbers come from is Deep Dive Q2 (§9, Option D): a background job runs self-hosted routing (OSRM/Valhalla) with time-of-day traffic profiles to generate drive-time **isochrones** around each of the 500 DCs every 5–15 minutes, then rasterizes those isochrone contours onto the H3 grid and writes the result into this table. So step 3 is cheap specifically *because* the expensive part — actually simulating road-network drive time under current traffic — already happened offline, on a schedule, for every cell in advance; the request just reads the answer.

**Step 4 — add live `prep_time[dc]` → keep DCs with total ETA ≤ 60 min (and `status ≠ CLOSED`).** Step 3 only gave *drive* time, which is the slow-changing half of the promise (traffic shifts over ~15 minutes, so it's fine to precompute). The fast-changing half — how backed up a DC's picking queue is right now, and whether a driver is even available — can't be precomputed on the same schedule, so it's fetched live: the Fulfillment Service publishes a rolling `prep_time[dc]` (p50/p90 of recent pick-to-pack duration) roughly every 30 seconds, and this step adds that to each candidate DC's drive time from step 3 to get `ETA(cell, dc, t) = drive_time + prep_time + dispatch_wait` (the full formula is in §9). Any DC whose total ETA exceeds 60 minutes, or whose `status` is `CLOSED`, gets dropped from the candidate list right here — so the bitmap merge in step 5 only ever OR's together DCs that can actually fulfill the order in time. Using **p90** rather p50 for this threshold (as §9 recommends) matters because the 60-minute figure is a customer-facing promise, not an average — and this live step is also what makes an overloaded DC "self-correct": as its queue grows, `prep_time` grows, its ETA crosses 60 minutes, and it silently drops out of the eligible set for distant cells without any special-case logic anywhere else in the system.

### Flow B — Place order (FR2, write path)

```
1. Client POST /orders with Idempotency-Key
2. Order Service checks idempotency table → if seen, return stored response
3. Resolve delivery cell → eligible DCs (ETA ≤ 60) → choose ONE DC that can fulfill all lines
4. Single DB transaction on that DC's inventory shard:
     conditional-decrement available for every line (sorted by item_id)
     INSERT order (status = RESERVED, expires_at = now() + 5 min)
     INSERT outbox event "OrderReserved"
   COMMIT  → any line fails ⇒ ROLLBACK ⇒ 409 with unavailable items
5. Authorize payment (outside the DB transaction, with provider idempotency key = order_id)
6. CAS status RESERVED → CONFIRMED  (or RESERVED → CANCELLED + release on decline)
7. Outbox relay → Kafka "order-events" → Fulfillment/Dispatch, Notifications
8. Return 201 CONFIRMED
```

### Flow C — Inventory change propagation (FR3 → FR1)

```
1. Order reserve / restock / shrink commits to Inventory DB (Postgres)
2. CDC (Debezium on WAL) → Kafka "inventory-changes" (partitioned by dc_id)
3. Availability Projector consumes → recomputes item's stock bucket
4. Only if bucket changed (e.g., IN_STOCK → LOW → OUT): publish delta
5. Every Availability API node applies delta to its in-memory bitmap snapshot (+ Redis copy)
   Typical end-to-end lag: ~1 s
```

### Flow D — Fulfillment & tracking (FR4)

```
1. Fulfillment Service consumes OrderConfirmed → creates pick task at DC
2. Picker scans items → on_hand -= qty, reserved -= qty (atomic) → status PICKING → PACKED
3. Dispatch Service assigns nearest available driver (Redis GEO) → OUT_FOR_DELIVERY
4. Driver app streams location → Kafka → Tracking Service → SSE/WebSocket to customer
5. Delivered → capture payment → DELIVERED
```

---

## 7. High-Level Design

```
                        ┌───────────────────────────────────────────────┐
                        │   Clients: Customer app/web · DC handhelds    │
                        │            · Driver app                       │
                        └──────────────────────┬────────────────────────┘
                                               │
                                ┌──────────────▼──────────────┐
                                │ CDN (catalog images/static) │
                                │ API Gateway / LB            │
                                │ auth · rate limit · routing │
                                └─┬──────────┬──────────┬─────┘
                ┌─────────────────┘          │          └─────────────────────┐
                │ FR1                        │ FR2                            │ FR3 / FR4
   ┌────────────▼────────────┐  ┌────────────▼────────────┐   ┌───────────────▼──────────────┐
   │  Availability Service   │  │     Order Service       │   │  Inventory Service (WMS API) │
   │  (stateless, autoscale) │  │  idempotency · DC pick  │   │  restock · pick · shrink     │
   │  in-memory:             │  │  reserve txn · saga     │   ├──────────────────────────────┤
   │   • bitmap snapshot     │  └──┬─────────┬─────────┬──┘   │  Fulfillment Service         │
   │   • cell→DC table       │     │         │         │      │  pick tasks · state machine  │
   └──┬──────┬──────┬────────┘     │         │         │      ├──────────────────────────────┤
      │      │      │              │         │         │      │  Dispatch + Tracking Service │
      │      │      │              │  ┌──────▼──────┐  │      │  driver assign · live ETA    │
      │      │      │              │  │  Payment    │  │      └───┬───────────┬──────────┬───┘
      │      │      │              │  │  Service ───┼──┼──▶ Stripe/Adyen     │          │
      │      │      │              │  └─────────────┘  │          │           │          │
      │      │      │   ┌──────────▼───────────────────▼──────────▼──┐  ┌─────▼────┐ ┌───▼──────┐
      │      │      │   │  Inventory + Orders DB (PostgreSQL)        │  │ Redis GEO│ │ WebSocket│
      │      │      │   │  sharded by region (dc_id → shard)         │  │ (drivers)│ │ / SSE    │
      │      │      │   │  primary + sync standby + read replicas    │  └──────────┘ │ gateway  │
      │      │      │   │  tables: inventory, orders, order_lines,   │               └──────────┘
      │      │      │   │  inventory_events, idempotency, outbox     │
      │      │      │   └───────────────────┬────────────────────────┘
      │      │      │                       │ WAL → CDC (Debezium) + Outbox relay
      │      │      │          ┌────────────▼──────────────────────────────┐
      │      │      │          │  Kafka                                    │
      │      │      │          │  inventory-changes (key=dc_id)            │
      │      │      │          │  order-events      (key=order_id)         │
      │      │      │          │  driver-locations  (key=driver_id)        │
      │      │      │          └───┬───────────────────────┬───────────────┘
      │      │      │   ┌──────────▼──────────┐   ┌────────▼──────────────┐
      │      │      │   │ Availability        │   │ Notification Service  │
      │      │      │   │ Projector           │   │ push / SMS / email    │
      │      │      │   │ bucket-change only  │   └───────────────────────┘
      │      │      │   └──────────┬──────────┘
      │      │      │              │ deltas (pub/sub) + snapshot to Redis
      │      │      └──────────────┘
      │      │
      │  ┌───▼──────────────────┐     ┌───────────────────────────────────────┐
      │  │ Redis Cluster        │◀────│ ETA Precompute Job (every 5–15 min)   │
      │  │ • availability snaps │     │ OSRM/Valhalla + traffic profiles      │
      │  │ • cell→DC ETA table  │     │ isochrones per DC → rasterize to H3   │
      │  │ • catalog cache      │     └───────────────────────────────────────┘
      │  └──────────────────────┘
  ┌───▼────────────────────┐
  │ Elasticsearch          │
  │ catalog text search    │
  │ (global, not per-DC)   │
  └────────────────────────┘
```

### How each functional requirement is covered

| FR | Path through the design |
|---|---|
| **FR1 Browse availability** | Gateway → Availability Service → (cell→DC table + live prep time) → OR-merge in-memory bitmaps → ES catalog search filtered by bitmap → response. Never touches the transactional DB on the hot path. |
| **FR2 Place order** | Gateway → Order Service → idempotency check → choose single DC → one Postgres transaction with conditional decrements + order row + outbox → payment auth → CAS to CONFIRMED → Kafka order-events. |
| **FR3 Inventory mgmt** | Handheld/WMS → Inventory Service → Postgres (inventory + ledger in same txn) → CDC → Kafka → Projector → availability snapshots. |
| **FR4 Fulfillment & tracking** | order-events → Fulfillment (pick tasks; consumes `reserved`) → Dispatch (Redis GEO nearest driver) → driver-locations → Tracking → WebSocket/SSE to customer; Notification Service for push/SMS. |

### Per-component "why"

| Component | Choice | Why |
|---|---|---|
| Inventory + Orders DB | **PostgreSQL** | Multi-row ACID transactions, conditional updates, CHECK constraints, mature ops. Volume (600 decrements/s peak) is tiny for Postgres. Orders and inventory for a DC live **on the same shard** → no distributed transaction. |
| Availability read model | **In-memory bitmaps + Redis** | Total state ≈ 1 MB → replicate to every API node; zero network hops on the hot path. Redis is the warm-start source and fallback. |
| Change propagation | **CDC → Kafka** | No dual-write: the DB commit is the event. Partition by `dc_id` preserves per-DC ordering. Replayable for rebuilding projections. |
| Order events | **Transactional outbox → Kafka** | Order row and event are committed atomically; relay publishes at-least-once; consumers are idempotent by `order_id`. |
| Catalog search | **Elasticsearch** | Relevance, typo tolerance, facets. Catalog is global and slow-changing — don't index per-DC availability into it (reindex churn). |
| Geo/ETA | **H3 cells + precomputed isochrones** | Turns an expensive routing call into an O(1) lookup. |
| Driver location | **Redis GEO + Kafka** | High write rate, ephemeral, AP is fine. |

---

## 8. Deep Dive Q1: Availability Within a Fixed Distance (30 miles)

**Question:** How will customers be able to query the availability of items within a fixed distance (e.g., 30 miles)?

### The problem decomposes into two steps
1. **Which DCs are within 30 miles of the customer?** (geo problem)
2. **Which items are in stock at any of those DCs?** (inventory aggregation problem)

### Step 1 options — finding nearby DCs

| Option | How | Pros | Cons |
|---|---|---|---|
| **A. Scan all DCs (Haversine) in memory** | Keep all 500 DCs in an array; compute great-circle distance for each | Trivial; 500 × ~50 ns ≈ 25 µs; no infra | Doesn't scale to 1M points (not a concern at 500) |
| **B. PostGIS `ST_DWithin` on a GiST index** | `SELECT dc_id FROM dcs WHERE ST_DWithin(geog, :pt, 48280)` | Exact, standard, handles polygons | DB round-trip on the hot path for data that never changes |
| **C. Geohash prefix** | Query the customer's geohash + 8 neighbors at a precision covering 30 mi | Works in any KV store | Edge effects; cell size mismatch with a fixed radius |
| **D. Quadtree / R-tree** | Spatial index in memory | Great for millions of moving points | Overkill for 500 static points |
| **E. H3 cell → DC precompute** | Map every H3 cell to the DCs within 30 mi offline | O(1) lookup; extends naturally to ETA (Q2) | Precompute job + storage (~72 MB) |

**Recommendation:**
- For pure 30-mile radius: **Option A** — say explicitly that with 500 DCs a spatial index is unnecessary. Staff candidates resist reflexively reaching for geohash/quadtree (see the Proximity Service guide for when those *are* needed).
- Design the interface around **Option E** anyway, because Q2 (1-hour ETA) needs a precomputed cell → DC mapping, and the same lookup serves both.

### Step 2 options — aggregating inventory across those DCs

| Option | How | Pros | Cons |
|---|---|---|---|
| **A. SQL query on the primary** | `SELECT item_id, SUM(on_hand - reserved) FROM inventory WHERE dc_id IN (...) GROUP BY item_id HAVING SUM(...) > 0` | Always correct | Puts browse load (6–12K QPS) on the OLTP DB that orders depend on |
| **B. SQL on read replicas** | Same query, routed to replicas | Isolates order path; seconds of lag acceptable | Still a 10K-row aggregation per request; replica fleet needed |
| **C. Redis per-DC hash `inv:{dc}` → item→qty** | HGETALL per DC, merge in app | Fast, simple | 3 DCs × 10K fields per request = heavy payloads |
| **D. Per-DC availability bitmap, OR-merged** ✅ | 1 bit per SKU per DC; union = bitwise OR | ~1.25 KB per DC; OR of 3 bitmaps is nanoseconds; fits in every node's RAM | Loses exact quantity (use 2-bit buckets for "low stock") |

### Recommended answer (bullets for the interview)
- Snap the customer's `lat/lng` to an **H3 res-8 cell** (~0.74 km²).
- Look up **DCs within 30 miles** — either Haversine over the 500 in-memory DCs or the precomputed `cell → [dc_id]` table. Filter out `CLOSED` DCs.
- Fetch each candidate DC's **availability bitmap** from the in-memory snapshot (bit *i* = SKU index *i* has `available > 0`).
- **Bitwise-OR** the bitmaps → "available somewhere reachable" set.
- If the request has a search term/category, query **Elasticsearch** for ranked candidate SKUs, then **filter by the merged bitmap**; page the results.
- Hydrate names/prices/images from the **catalog cache** (CDN for images).
- Return with an **`as_of` timestamp** — the result is eventually consistent; the authoritative check happens at order time (Q4).

### Staff trap: union availability vs single-DC fulfillment
- If item A is only at DC1 and item B only at DC2, the union shows both — but a single-DC order can't ship both.
- Options:
  - **Show only the primary (best-ETA) DC's stock** → consistent with fulfillment, simple; but hides items another DC could deliver.
  - **Show the union, validate in the cart** → richer catalog; cart warns "these items come from different locations" → user removes or accepts a split delivery.
  - **Hybrid (recommended):** browse from the primary DC; if an item is out there but in stock at a secondary DC within the ETA bound, show it with a badge ("delivered separately" or slightly later ETA). Cart computes the fulfillment plan before checkout.

---

## 9. Deep Dive Q2: Only Items Deliverable Within 1 Hour

**Question:** How can we extend the design so that we only return items that can be delivered within 1 hour?

### Why distance isn't enough
- 30 miles across a bridge at 5:30 pm ≠ 30 miles of rural highway at 10 am.
- Delivery promise = **drive time** (traffic- and time-of-day dependent) + **DC prep time** (pick/pack queue) + **dispatch wait** (driver availability).
- `ETA(cell, dc, t) = drive_time(dc → cell, t) + prep_time(dc, t) + dispatch_wait(dc, t)` — eligible iff `ETA ≤ 60 min`.

### Options

| Option | How | Pros | Cons |
|---|---|---|---|
| **A. Live routing call per request** | For each candidate DC, call Google Distance Matrix / Mapbox on every browse | Most accurate (live traffic) | 12K QPS × 3–5 DCs = 50K external calls/s — cost-prohibitive, adds 100–300 ms, external dependency on the hot path |
| **B. Straight-line heuristic** | Assume avg speed (e.g., 25 mph) → radius ≈ 20 mi minus prep | Free, instant | Wrong at rivers/bridges/mountains; over-promises in traffic |
| **C. Static service polygons per DC** | Ops draws a polygon per DC; point-in-polygon | Fast; ops control | Doesn't adapt to traffic, weather, DC load; manual upkeep |
| **D. Precomputed isochrones → H3 table, refreshed every 5–15 min** ✅ | Self-hosted OSRM/Valhalla with time-of-day traffic profiles generates drive-time contours per DC; rasterize to H3 cells → `cell → [(dc, drive_secs)]` | O(1) lookup; traffic-aware; no external call on hot path; cacheable per cell | Freshness bounded by refresh interval; precompute infra |
| **E. ML ETA model** | Model trained on historical delivery times by cell/DC/hour/weather | Captures real-world effects (parking, apartments, elevators) | Needs data and MLOps; use to *calibrate* D, not replace it |

### Recommended answer (bullets for the interview)
- **Separate the slow-changing part from the fast-changing part:**
  - **Drive time** changes with traffic over ~15 min → **precompute**.
  - **Prep time & driver availability** change minute-to-minute → **apply live at query time**.
- **Precompute job (every 5–15 min, per traffic profile):**
  - For each DC, generate **isochrones** (e.g., 10/20/30/40/50-minute contours) using self-hosted **OSRM/Valhalla** with time-of-day speed profiles.
  - Rasterize contours to **H3 res-8 cells** → store `cell → [(dc_id, drive_secs)]` sorted by drive time (~72 MB) in Redis and replicate in memory.
  - Isochrones are cheap: 500 DCs × a handful of contours per cycle, instead of millions of point-to-point calls.
- **At query time:**
  - Look up the customer's cell → candidate DCs with `drive_secs`.
  - Add **live `prep_time[dc]`** (rolling p50/p90 of pick-to-pack time, published by the Fulfillment Service every ~30 s) and **dispatch wait** (from driver supply near the DC).
  - Keep DCs where total ETA ≤ 60 min (use **p90**, not p50, if the promise is a guarantee).
  - OR-merge only those DCs' bitmaps → items deliverable within 1 hour.
- **DC load as backpressure:** when a DC's queue grows, its `prep_time` grows → it **automatically drops out** of far cells' eligible set. Physical capacity becomes visible in the catalog without special-case code.
- **Fallbacks:** if the precompute job fails, keep serving the last-good table (with a staleness alarm); if that's missing, fall back to Option B with a conservative radius.
- **At order time:** recompute ETA with the live values for the chosen DC; reject if it no longer meets 60 min (rare, e.g., the DC just went BUSY).
- **Calibration loop:** compare promised vs actual delivery times per cell/DC → adjust buffers (Option E as a correction layer).

---

## 10. Deep Dive Q3: Placing Orders

**Question:** How will customers be able to place orders?

### Recommended answer (bullets for the interview)
- **Client side:** cart is client/session state (not reserved). On "Place order," the client generates an **`Idempotency-Key`** (UUID) once per checkout attempt and retries with the same key.
- **Gateway:** authenticates, rate-limits per user (anti-bot/hoarding), routes to Order Service.
- **Order Service step 1 — idempotency:** `INSERT INTO idempotency_keys (key, customer_id, status='IN_PROGRESS')` — on conflict, return the stored response (or 409 "in progress").
- **Step 2 — validate:** item IDs exist, prices match `expected_total_cents`, age-restricted items require verified ID, address is serviceable.
- **Step 3 — choose fulfilling DC:** from the eligible DCs (ETA ≤ 60), pick the one with the best ETA that **has all lines** (check against the bitmap first, authoritative check in step 4). If none can fulfill everything: return 409 with which items are the problem (or offer a split — see options below).
- **Step 4 — reserve inventory atomically** (detailed in Q4): one transaction on that DC's shard that conditionally decrements availability for all lines, inserts the order as `RESERVED` with `reservation_expires_at = now() + 5 min`, and writes an outbox row. All-or-nothing.
- **Step 5 — authorize payment** *after* commit, outside any DB transaction (never hold row locks across a network call). Use `order_id` as the payment provider's idempotency key.
- **Step 6 — finalize with a CAS state transition:**
  - Success: `UPDATE orders SET status='CONFIRMED' WHERE order_id=? AND status='RESERVED'`.
  - Decline: `status='CANCELLED'` + release reservations in the same transaction.
  - If the CAS fails (the expiry sweeper already cancelled it), **void the authorization** — see the race in §11.
- **Step 7 — publish:** the outbox relay emits `OrderConfirmed` → Fulfillment creates a pick task; Notification Service sends confirmation; client gets `201 CONFIRMED` with ETA.
- **Capture** the payment on delivery (or at pick time for partial-fulfillment adjustments).

### Order state machine

```
              reserve OK                 payment auth OK
  (new) ───────────────▶ RESERVED ───────────────────────▶ CONFIRMED ──▶ PICKING ──▶ PACKED
     │                     │   │                               │                       │
     │ stock fail (409)    │   │ auth declined / expired       │ cancel (pre-pick)     ▼
     ▼                     │   ▼                               ▼               OUT_FOR_DELIVERY
  REJECTED                 │  CANCELLED ◀──────────────────────┘                       │
                           │  (release reserved)                                       ▼
                           └──────────────────────────────────────────────────▶   DELIVERED
                                                                                (capture payment)
  Every transition = conditional UPDATE ... WHERE status = <expected>  (CAS)
```

### Options for orchestrating checkout

| Option | Pros | Cons |
|---|---|---|
| **Single synchronous request (reserve → auth → confirm)** ✅ | Simple; user gets a definitive answer in < 1 s; DB work is one local transaction | Payment provider latency is on the request path |
| **Async order (accept → queue → process → notify)** | Absorbs huge spikes; decouples from payment latency | User doesn't know immediately whether it worked — poor UX for a 1-hour promise; harder failure messaging |
| **Orchestrated saga (Temporal/Step Functions)** | Durable retries, visible workflow state, clean compensation | More infra; overkill unless checkout spans many services (promos, loyalty, fraud) |
| **Payment first, then reserve** | Never reserve for users who can't pay | Must refund when stock fails — customers see charges appear and vanish; worse than reserving first |

**Recommendation:** synchronous reserve → authorize → confirm, with the **reservation TTL** as the compensation safety net. Move to a saga engine when checkout grows more steps.

### Extension: multi-DC split orders
- **Reject and suggest** (default): simplest, one driver.
- **Split into child orders**, each a separate single-DC transaction; reserve sequentially with compensation (release earlier reservations if a later one fails). Costs a second driver → only offer when the business accepts the margin hit.

---

## 11. Deep Dive Q4: Preventing Oversell & Race Conditions

**Question:** How can we ensure users cannot order items that are unavailable? How do we avoid race conditions?

### The race

```
Stock of sku_42 at dc_118 = 1

T1 (Alice)                         T2 (Bob)
SELECT available → 1               SELECT available → 1
  "OK, in stock"                     "OK, in stock"
UPDATE available = 0               UPDATE available = 0
COMMIT  ✅                          COMMIT  ✅      ← both sold the same unit (lost update)
```

The browse-time availability (eventually consistent) can never be the guarantee — the guarantee must come from an **atomic check-and-decrement at the source of truth**.

### Options

| Option | How | Pros | Cons |
|---|---|---|---|
| **A. Check then write (no locking)** | `SELECT`, compare in app, `UPDATE` | Simple | **Broken** under concurrency (above) |
| **B. Pessimistic lock** | `SELECT ... FOR UPDATE` on each inventory row, check, update | Correct; intuitive | Holds locks longer; **deadlocks** if two orders lock rows in different order → must sort by `item_id` |
| **C. Atomic conditional update** ✅ | `UPDATE inventory SET reserved = reserved + :q WHERE dc_id=:dc AND item_id=:i AND on_hand - reserved - safety_stock >= :q` → check rowcount = 1 | Correct in one statement; shortest lock hold; no read-modify-write window | Must check each row count and roll back on any miss |
| **D. Optimistic concurrency (version column)** | Read version, `UPDATE ... WHERE version = :v`, retry on conflict | No locks held while thinking | Retries storm under contention on hot SKUs |
| **E. SERIALIZABLE isolation** | Run A inside SERIALIZABLE; DB aborts conflicting txns | Correct by construction | Abort/retry overhead; easy to misuse; harder to reason about performance |
| **F. Redis Lua atomic decrement** | Lua script checks and decrements N keys atomically | Very fast (100K+ ops/s); good for flash sales | Async replication → a failover can **lose acknowledged decrements → oversell**; dual-write with the DB; not a system of record |
| **G. Distributed lock (Redlock) per SKU** | Acquire lock, check, update, release | Familiar pattern | Lock-service failure/expiry semantics are subtle (needs fencing tokens); adds a hop; strictly worse than letting the DB do it |
| **H. DynamoDB `TransactWriteItems` + `ConditionExpression`** | Conditional decrement on up to 100 items atomically | Managed, scales, same semantics as C | Transaction cost (2× WCU), item limits; fine alternative if already on DynamoDB |

### Recommended answer (bullets for the interview)
- **Never trust browse-time availability.** It's a projection with ~1 s lag; it's a hint, not a promise.
- **Make the inventory DB the single source of truth**, and enforce availability with an **atomic conditional update** inside one transaction:

```sql
BEGIN;  -- READ COMMITTED is sufficient for this pattern in Postgres

-- One statement per line, ORDERED BY item_id to make lock acquisition order deterministic
UPDATE inventory
   SET reserved = reserved + :qty, version = version + 1, updated_at = now()
 WHERE dc_id = :dc AND item_id = :item
   AND on_hand - reserved - safety_stock >= :qty;
-- app checks rowcount == 1 for EVERY line; any 0 → ROLLBACK → 409 with the failing items

INSERT INTO orders (order_id, customer_id, dc_id, status, reservation_expires_at, idempotency_key, ...)
VALUES (:id, :cust, :dc, 'RESERVED', now() + interval '5 minutes', :ikey, ...);

INSERT INTO order_lines (...) VALUES (...);
INSERT INTO inventory_events (dc_id, item_id, delta, reason, ref_id) VALUES (..., 'RESERVE', :id);
INSERT INTO outbox (aggregate_id, type, payload) VALUES (:id, 'OrderReserved', ...);
COMMIT;
```

- **Why this is race-free:** in Postgres, an `UPDATE`'s `WHERE` clause is **re-evaluated after acquiring the row lock** (if another txn changed the row first, the predicate is re-checked against the new version). Two concurrent orders for the last unit serialize on the row lock; the second sees `available = 0` and matches 0 rows.
- **Defense in depth:** `CHECK (reserved <= on_hand AND reserved >= 0)` constraint — even a buggy code path can't commit an oversell.
- **All-or-nothing across lines:** all lines are in one local transaction because **an order is fulfilled from one DC, and inventory + orders for a DC live on the same shard** → no 2PC, no saga for the reservation itself.
- **Deadlock avoidance:** always update lines **sorted by `item_id`**; two orders sharing SKUs acquire locks in the same order.
- **Short transactions:** no network calls (payment, ETA) inside the transaction; lock hold time is a few milliseconds.
- **Duplicate submissions:** `UNIQUE (idempotency_key)` on orders + idempotency table → a retried request can't reserve twice.
- **Reservation expiry (abandoned/failed payments):** a sweeper releases holds:

```sql
-- runs every ~10 s; SKIP LOCKED lets multiple sweeper instances work in parallel safely
WITH expired AS (
  SELECT order_id FROM orders
   WHERE status = 'RESERVED' AND reservation_expires_at < now()
   FOR UPDATE SKIP LOCKED LIMIT 500)
UPDATE orders SET status = 'CANCELLED' WHERE order_id IN (SELECT order_id FROM expired);
-- + decrement inventory.reserved for their lines, in the same transaction
```

- **The payment-vs-sweeper race (Staff trap):** payment auth succeeds at 4:59.9 while the sweeper cancels at 5:00.0.
  - Both use **CAS on status**: only one of `RESERVED → CONFIRMED` / `RESERVED → CANCELLED` can win.
  - If the confirm CAS loses → **void the authorization** and tell the user to retry. Never confirm an order whose stock was released.
  - Use **DB time (`now()`)** for expiry, never app-server clocks (clock skew).
- **Hot-SKU contention:** at ~1 order/s per DC, row-lock contention is negligible. For flash drops (e.g., a limited-edition item), add an **admission gate** in Redis (token bucket = stock count) in front of the DB so only N contenders reach the row lock; the DB still makes the final decision.
- **Physical reality ≠ system count:** `safety_stock` buffers mis-shelving and damage; when a picker can't find an item, fulfillment triggers **substitution / partial refund** and an inventory `ADJUST` event. Software prevents *logical* oversell; process handles *physical* discrepancy.

---

## 12. Deep Dive Q5: Fast & Highly Available Availability Lookups

**Question:** How can we ensure availability lookups are fast and available?

### Options

| Option | Latency | Availability | Freshness | Notes |
|---|---|---|---|---|
| **A. Query primary DB** | 10–50 ms (aggregation) | Coupled to order path | Exact | Browse spikes would starve checkout — reject |
| **B. Read replicas** | 10–50 ms | Good | ~100 ms–s lag | Replica fleet sized for 12K QPS aggregations; still a network hop |
| **C. Redis cache, TTL-based (cache-aside)** | 1–2 ms | Good | Up to TTL stale | Cache stampede on expiry for popular cells; TTL trade-off |
| **D. Redis cache, event-driven (CDC) updates** | 1–2 ms | Good | ~1 s | No TTL guessing; needs CDC pipeline |
| **E. In-memory bitmap snapshot on every API node, fed by CDC deltas** ✅ | **< 1 ms** (no network hop) | **Best** — survives Redis/DB outage with last-known state | ~1 s | Possible only because total state ≈ 1 MB |
| **F. Per-cell response cache (CDN/edge)** | ~10 ms at edge | Best | Short TTL (15–30 s) | Response depends only on (cell, filters) → shareable; great for home/category pages |

### Recommended answer (bullets for the interview)
- **CQRS: separate read model from write model.** Browse never touches the transactional DB; orders always do.
- **Compact representation:** per DC, a bitmap of SKU indices (1 bit, or 2-bit buckets for OUT/LOW/MED/PLENTY). 500 DCs ≈ **625 KB–1.25 MB total** → **replicate the whole thing into every Availability Service node's memory.**
- **Fast path per request:** cell → eligible DCs (in-memory table) → OR 2–5 bitmaps (nanoseconds) → filter catalog → respond. p99 dominated by search + serialization, not data access.
- **Freshness via CDC, not TTL:**
  - Postgres WAL → **Debezium** → Kafka `inventory-changes` (key = `dc_id`, preserving per-DC order).
  - **Availability Projector** maintains exact counts but **emits only bucket transitions** (e.g., 12 → 11 changes nothing; 1 → 0 flips a bit). This cuts fan-out traffic by orders of magnitude.
  - Nodes subscribe to deltas (Kafka consumer group per node, or Redis pub/sub) and apply them in place; each delta carries a per-DC **sequence number** so nodes detect gaps and resync.
- **Warm start & recovery:** projector writes a versioned full snapshot to Redis every ~30 s; a new node loads the snapshot, then replays deltas from that sequence number.
- **Edge caching:** cache the response for `(h3_cell, category, page)` for 15–30 s at the CDN/gateway; popular cells in dense metros serve most traffic from cache. Personalized bits (cart, favorites) come from a separate call.
- **Catalog separately:** names, prices, images change rarely → CDN + long-TTL cache; Elasticsearch for text search only.
- **Graceful degradation (availability over freshness):**
  - Redis down → nodes keep serving their in-memory snapshot.
  - Kafka/CDC lagging → keep serving, set `as_of`, alert at lag > 30 s; optionally mark LOW items as OUT conservatively.
  - ETA precompute stale → last-good table, then conservative radius fallback.
  - The order path's authoritative check (Q4) catches every stale read — so stale browse data costs a 409 + suggestions, never an oversell.
- **Multi-AZ / multi-region:** stateless Availability Service in ≥ 3 AZs behind the LB; Redis cluster with replicas; Kafka RF = 3, `min.insync.replicas = 2`, `acks = all` for inventory-changes.
- **Monitor the read model's correctness, not just its latency:** sample-compare snapshot vs DB per DC every minute; alert on divergence; track "browse-showed-available but order rejected" rate as the user-facing freshness SLI.

---

## 13. NFR Deep Dive 1: Strong Consistency for Inventory (No Oversell)

### Goal
Zero software-caused oversells; no acknowledged order lost.

### How it's achieved

| Mechanism | What it guarantees |
|---|---|
| **Single source of truth** (Postgres inventory table) | One authority for "how many can be sold" |
| **Atomic conditional update** (§11) | No read-modify-write window → no lost updates |
| **Single-DC orders co-located on one shard** | All lines reserved in one local ACID transaction — no 2PC, no partial reservation |
| **CHECK constraints** | Invariant `0 ≤ reserved ≤ on_hand` enforced by the DB, not just app code |
| **Deterministic lock ordering** | No deadlocks between concurrent multi-line orders |
| **Idempotency keys** (client → order, order → payment) | Retries can't double-reserve or double-charge |
| **CAS state transitions** | Exactly one of confirm/cancel wins; no confirmed order with released stock |
| **Reservation TTL + sweeper (SKIP LOCKED)** | Abandoned checkouts can't strand stock |
| **Transactional outbox** | Order row and its event commit atomically → downstream never misses or invents an order |
| **Synchronous replication to a standby** (`synchronous_commit = on` with one sync standby in another AZ) | RPO = 0 for acknowledged orders on primary failure |
| **Inventory event ledger** | Every balance change auditable; nightly reconciliation of `on_hand` vs `SUM(ledger)` |

### Choosing the consistency store — options

| Option | Pros | Cons |
|---|---|---|
| **PostgreSQL sharded by region** ✅ | ACID, conditional updates, constraints, team familiarity, cheap at this volume | Manual sharding; failover takes ~10–30 s |
| **Google Spanner / CockroachDB** | Global serializable transactions, automatic sharding, multi-region | Higher latency (consensus per write), cost, vendor lock-in — unnecessary when orders are region-local |
| **DynamoDB with transactions** | Managed, elastic, conditional writes | 100-item transaction limit (fine), 2× cost for transactions, weaker ad-hoc querying for ops |
| **Redis as inventory of record** | Fastest | Async replication can lose writes on failover → oversell; reject as system of record |

### Consistency boundaries — be explicit

```
STRONG (CP):   inventory reservation · order state transitions · payment idempotency
EVENTUAL (AP): browse availability · catalog · ETA table · tracking map · notifications
Bridge:        CDC + outbox; order-time re-check catches every stale read
```

### During a partition / primary failover
- Writes to the affected region's shard **fail closed** (return 503 "try again in a moment") for the failover window — refusing orders is better than overselling.
- Browse continues from in-memory snapshots (AP).
- Other regions are unaffected (blast radius = one shard).
- Use a **fencing token / epoch** (Patroni/etcd leader key) so a zombie old primary can't accept writes after failover.

---

## 14. NFR Deep Dive 2: Low-Latency, Highly Available Reads

### Latency budget (p99 server-side < 100 ms)

```
Gateway auth + routing ..................  5 ms
H3 snap + cell→DC lookup (in-memory) ....  < 1 ms
Live prep_time filter (in-memory) .......  < 1 ms
Bitmap OR (in-memory) ...................  < 0.1 ms
Elasticsearch catalog query .............  15–40 ms   ← dominant cost
Bitmap filter + hydrate (catalog cache) .  2–5 ms
Serialization ...........................  2–5 ms
────────────────────────────────────────────────────
Total ...................................  ~30–60 ms  (headroom for tail)
Category/home pages without q: skip ES → precomputed per-category SKU lists → ~10 ms
```

### How it's achieved
- **No network hop for availability data** — the data structure is small enough to live in every process (§12).
- **Precompute what's expensive** — ETA isochrones offline; per-category item rankings offline.
- **Cache by location cell, not by user** — high hit ratio in dense areas; responses are shareable.
- **Edge cache** for anonymous/home/category responses (15–30 s TTL); stale-while-revalidate to avoid stampedes.
- **Request coalescing** in the gateway for identical concurrent cache misses.
- **Hedged requests** to Elasticsearch replicas for tail latency (send a second request after p95).
- **Bulkheads:** separate thread pools/connection pools for browse vs checkout so a slow ES never consumes checkout capacity.

### Availability (99.99% for browse)
- Stateless Availability Service, ≥ 3 AZs, autoscaled on CPU/QPS.
- **Every dependency has a degraded mode:**

| Dependency down | Degraded behavior |
|---|---|
| Elasticsearch | Fall back to precomputed category lists + prefix match on in-memory catalog names |
| Redis | Serve in-memory snapshot; new nodes boot from S3 snapshot copy |
| Kafka / CDC | Serve last-known snapshot with `as_of`; alert on staleness |
| ETA precompute | Last-good table; then conservative radius heuristic |
| Inventory DB | Browse unaffected (doesn't touch it); checkout fails closed |

- **Circuit breakers** around ES and Redis (see Circuit Breaker guide) — fail fast into the degraded path rather than queueing.

---

## 15. NFR Deep Dive 3: Scalability Under Demand Spikes

### What spikes look like
- Daily: evening and late-night peaks (5×).
- Events: Super Bowl, finals, storms/snow days, holiday eves (10×+), concentrated in specific metros.
- Spikes are **geographically correlated** — one metro's DCs get hammered while others are idle.

### Layer-by-layer strategy

| Layer | Bottleneck | Strategy |
|---|---|---|
| **Browse (reads)** | CPU on API nodes, ES | Horizontal autoscale (stateless); in-memory data means linear scaling; edge cache absorbs repeat cells; pre-scale before known events (scheduled scaling) |
| **Catalog search** | ES query throughput | Add replicas; cache top queries per cell; degrade to category lists |
| **Order placement** | Postgres write throughput per shard | 120 orders/s ≈ 600 row updates/s is far below one Postgres primary's capacity; shard by region for **blast radius** and to scale to 10×; PgBouncer for connection pooling |
| **Payment provider** | External rate limits/latency | Connection pools, timeouts, idempotent retries; circuit breaker → fail fast with "try again" rather than piling up |
| **Kafka** | Partition throughput | inventory-changes partitioned by `dc_id` (500 keys over e.g. 64 partitions); order-events by `order_id` |
| **Physical DC capacity** ⚠️ | Pickers, packing stations, drivers | **The real bottleneck.** Software must not accept more orders than a DC can fulfill within the promise (below) |

### Staff insight: the binding constraint is physical, not software
- You can scale servers in minutes; you can't add pickers or drivers in minutes.
- **Capacity-aware admission:**
  - Each DC publishes live `prep_time` and `orders_in_queue`; dispatch publishes driver supply per DC.
  - As queue depth grows, ETA grows → the DC **drops out of the 1-hour set for farther cells first** (Q2), shrinking its effective service area automatically.
  - At a hard threshold, mark the DC `BUSY` → only nearest cells served, or show "Delivery in ~75 min" (honest promise) instead of a broken one.
  - Optional: **dynamic delivery fee / surge** to shape demand (business decision — call it out, don't decide it).
- **Throttle before the DB, not at it:** per-DC order admission token bucket (rate = DC throughput) in the Order Service; requests beyond it get an honest "high demand" response immediately.

### Extreme spikes / bot traffic
- **Virtual waiting room** at the gateway for flash drops.
- Per-user and per-device rate limits on POST /orders; purchase limits per SKU per customer (anti-hoarding).
- **Load shedding priority:** checkout > cart > browse > recommendations — shed the cheapest-to-lose traffic first.

### Scaling the data layer when the business grows
- 500 → 5,000 DCs: bitmaps still ≈ 12.5 MB total → still in memory. Cell table grows ~10× → Redis, still fine.
- Orders 10×: add regional shards (consistent hashing over `dc_id` groups); orders and inventory stay co-located.
- New markets/countries: deploy as **independent regional cells** (full stack per region) — orders never cross regions, so there's no global transaction to coordinate.

---

## 16. CAP / PACELC Positioning

| Component | CAP | PACELC | Reasoning |
|---|---|---|---|
| Inventory reservation (Postgres) | **CP** | PC/EC | Overselling is worse than refusing an order; fail closed during failover |
| Order state transitions | **CP** | PC/EC | Exactly one of confirm/cancel must win; CAS on the primary |
| Payment (provider + idempotency) | **CP** | PC/EC | Money: idempotency keys, no double charge |
| Availability snapshot / bitmaps | **AP** | PA/EL | Stale by ~1 s is fine; order path re-checks |
| Cell → DC ETA table | **AP** | PA/EL | Minutes-stale traffic data is acceptable; last-good fallback |
| Catalog (ES, cache, CDN) | **AP** | PA/EL | Slow-changing; prices re-validated at checkout |
| Driver location / tracking | **AP** | PA/EL | Latest-wins, ephemeral; a missed ping is harmless |
| Notifications | **AP** | PA/EL | At-least-once with dedup by event id |
| Shard leadership (Patroni/etcd) | **CP** | PC/EC | Single writer per shard; fencing on failover |

**Pattern:** mostly AP/EL with **three named CP exceptions** — inventory, order state, payment — all on the write path.

---

## 17. Failure Modes & Mitigations

| Failure | Symptom | Detection | Mitigation |
|---|---|---|---|
| Concurrent orders for the last unit | Oversell | Reconciliation; CHECK constraint violations | Conditional atomic update; CHECK `reserved ≤ on_hand` |
| Payment succeeds after reservation expired | Charged, but stock released | CAS on confirm fails | Void auth; user retries; DB clock for expiry |
| Payment provider timeout (unknown outcome) | Order stuck in RESERVED | Timeout metrics | Retry with same idempotency key; query provider; reconcile job; sweeper releases if never authorized |
| Double-tap / client retry | Duplicate orders | Unique constraint hits | Idempotency-Key + UNIQUE index; return stored response |
| Deadlock between multi-line orders | Txn aborts | Postgres deadlock logs | Sort lines by `item_id`; short txns |
| Postgres primary failure | Checkout 5xx in one region | Health checks, Patroni | Sync standby promotion (RPO 0); fencing; fail closed; browse unaffected |
| CDC/Kafka lag | Browse shows stale stock → more 409s | Consumer lag, 409 rate | Alert; conservative LOW→OUT; scale projector; snapshot resync |
| Projector bug / missed delta | Snapshot drifts from DB | Sequence gaps; periodic DB sample compare | Gap detection → full resync from DB; snapshots versioned |
| ETA precompute job fails | Stale or missing reachability | Job freshness metric | Last-good table; conservative radius fallback |
| DC overwhelmed (snowstorm) | Promised ETAs missed | Promise-vs-actual SLI; queue depth | Live prep_time shrinks service area; BUSY status; per-DC admission throttle |
| DC outage (power/network) | Orders can't be picked | Heartbeat from DC systems | Mark DC CLOSED → drops from all cells; reroute open orders where possible; notify customers |
| Physical stock mismatch | Picker can't find item | Pick exceptions | Safety stock; substitution/partial refund; ADJUST event; cycle counts |
| Flash drop / bot surge on one SKU | Row-lock contention, unfair allocation | Hot-row metrics, per-user rate | Redis token-bucket gate; per-customer limits; waiting room |
| Elasticsearch outage | Search fails | Error rate, circuit breaker | Degrade to category lists / in-memory prefix search |
| Hot metro cache stampede | Burst of misses to ES | Miss rate spikes | Request coalescing; stale-while-revalidate |
| Zombie primary after failover | Split-brain writes | Epoch mismatch | Fencing tokens; old primary demoted/STONITH |

**Domain trap class (Staff signal):** "the system says available but the shelf is empty" is this domain's equivalent of the crawler's spider trap — a recurring correctness failure that comes from the gap between a model and reality. Name the class, then name the mitigations: safety stock, reconciliation, cycle counts, substitution flow, and measuring the "promised but not fulfilled" rate.

---

## 18. Scalability & Sharding

### Sharding key: `dc_id` (grouped by region)

```
shard_id = region_of(dc_id)          # e.g., ~10–20 regional shards, 25–50 DCs each
Each shard holds: inventory, orders, order_lines, inventory_events, outbox for its DCs
```

**Why `dc_id`:**
- An order touches exactly one DC → **every order transaction is single-shard**.
- Inventory and orders co-located → reservation + order insert in one ACID transaction.
- Regional spikes stay within a shard; blast radius of a failure = one region.

**What becomes expensive (name it — Staff signal):**
- "All orders for customer X" crosses shards → serve from a **secondary read model** (customer_orders projection from order-events into DynamoDB/Cassandra keyed by `customer_id`).
- Company-wide analytics → CDC into the warehouse (BigQuery/Snowflake/ClickHouse), never on OLTP shards.

### Options considered

| Shard key | Pros | Cons |
|---|---|---|
| **`dc_id` / region** ✅ | Single-shard orders; locality; blast radius | Customer history needs a projection; uneven regions → split hot regions |
| `customer_id` | Customer history is local | Inventory rows for one DC contended across all shards → distributed transactions for every order |
| `item_id` | Spreads hot SKUs | Multi-item orders span shards → 2PC — worst choice |
| `order_id` hash | Even distribution | Inventory not co-located → distributed transactions |

### Horizontal scaling plan

| Component | Strategy |
|---|---|
| Availability Service | Stateless; autoscale; data replicated in-process |
| Order Service | Stateless; autoscale; PgBouncer per shard |
| Postgres | Regional shards; sync standby + async read replicas; split a region when hot |
| Kafka | Add partitions; key by `dc_id` / `order_id` |
| Redis | Cluster mode; replicas per shard |
| Elasticsearch | Add replicas for read throughput |
| ETA precompute | Parallelize per DC; self-hosted routing cluster |

---

## 19. Senior vs Staff Differentiators

### Senior-level expectations
- Identify the main services: availability, orders, inventory, payments, fulfillment
- Geo lookup for nearby DCs (geohash/PostGIS)
- Use a transaction / locking to prevent overselling
- Cache availability in Redis
- Idempotent order placement
- Basic capacity math

### Staff-level additions

| Topic | What Staff adds |
|---|---|
| Framing | States the anchor constraint up front: **AP reads, CP writes, CQRS bridge** |
| Capacity | Notices the whole availability state is ~1 MB → in-process replication; avoids irrelevant storage sharding |
| Geo | Says "500 DCs don't need a spatial index"; uses H3 precompute because of the ETA requirement, not by reflex |
| 1-hour promise | Separates slow-changing drive time (precompute isochrones) from fast-changing prep/driver time (live) |
| Consistency | Conditional atomic update; explains Postgres re-evaluation of `WHERE` under row lock; CHECK constraint as defense in depth |
| Races | Payment-vs-expiry race resolved by CAS state transitions; void on lost race; DB clock |
| Sharding | Shard by `dc_id` *because* it makes every order single-shard; names the cross-shard query that becomes a projection |
| Union vs fulfillment | Calls out that union availability across DCs conflicts with single-DC fulfillment |
| Physical constraint | Recognizes pickers/drivers as the true bottleneck; ETA-driven backpressure and per-DC admission |
| Reality gap | Safety stock, reconciliation, substitution — software can't prevent physical shrinkage |
| Degradation | Every read dependency has a defined degraded mode; writes fail closed |

### The single most important Staff differentiator
**Separating the read model from the write model and saying exactly where consistency is enforced.** A Senior answer caches availability and then worries about cache staleness causing oversells. A Staff answer makes staleness *harmless*: the browse path is explicitly a hint, and the only promise is made by one atomic statement on the source of truth.

---

## 20. Interview Time Allocation

For a 45-minute session:

| Phase | Time | Focus |
|---|---|---|
| Requirements & scope | 4 min | 4 FRs, 3 NFRs; single-DC orders; state the AP-read/CP-write anchor |
| Capacity estimation | 3 min | 12K QPS browse, 120 orders/s, ~1 MB availability state |
| Entities + API | 4 min | Inventory (on_hand/reserved), Order state machine; Idempotency-Key |
| High-level design | 7 min | Boxes and arrows; walk FR1–FR4 through it |
| Deep dive: oversell & races (Q3/Q4) | 10 min | Conditional update, single-shard txn, TTL, CAS, idempotency |
| Deep dive: 30 mi / 1 hr availability (Q1/Q2) | 8 min | H3, isochrones, live prep time, union-vs-fulfillment |
| Deep dive: fast & available reads (Q5) | 5 min | Bitmaps in-process, CDC, edge cache, degraded modes |
| Failure modes + spikes | 4 min | Physical bottleneck, failover fail-closed, CDC lag |

**Cut if short:** FR4 tracking detail, split orders, ML ETA. **Never cut:** the race-condition answer and the read/write split.

---

## 21. Quick-Reference Cheatsheet

```
ANCHOR
──────
Browse = AP read model (bitmaps, ~1 s stale)  |  Order = CP write on source of truth
Stale browse costs a 409, never an oversell

KEY NUMBERS (illustrative)
──────────────────────────
10M DAU · 100M browse/day · ~1.2K avg / 12K spike QPS
1M orders/day · ~12/s avg · ~120/s peak · ~600 row updates/s peak
500 DCs × 10K SKUs = 5M inventory rows ≈ 500 MB
Availability bitmaps: 1.25 KB/DC · ~625 KB total → in every API node
H3 res 8 ≈ 0.74 km² · ~10K cells per 30-mi DC radius · cell table ≈ 72 MB

GEO / ETA
─────────
500 DCs → in-memory scan is enough for radius
1-hour: precompute isochrones (OSRM/Valhalla, traffic profiles) → H3 cell → [(dc, drive_secs)]
ETA = drive (precomputed) + prep (live) + dispatch wait (live) ≤ 60 min (use p90)
Busy DC → bigger prep time → shrinks service area automatically

NO-OVERSELL RECIPE
──────────────────
UPDATE inventory SET reserved = reserved + q
 WHERE dc=? AND item=? AND on_hand - reserved - safety_stock >= q   (rowcount must = 1)
All lines in ONE txn (single DC → single shard) · sort by item_id (no deadlocks)
CHECK (0 <= reserved <= on_hand) · UNIQUE idempotency_key
RESERVED (TTL 5 min) → auth payment outside txn → CAS to CONFIRMED
Sweeper: FOR UPDATE SKIP LOCKED · lost CAS → void auth · DB clock only

READ PATH
─────────
CDC (Debezium) → Kafka inventory-changes (key=dc_id) → Projector emits bucket flips only
→ in-memory bitmaps on every node (+ Redis snapshot, sequence numbers for gap detection)
Edge cache per (h3_cell, category) 15–30 s · ES for catalog text only

SYSTEMS
───────
Postgres (sharded by region/dc_id, sync standby) · Kafka · Redis · Elasticsearch
OSRM/Valhalla · H3 · Debezium · Outbox · Redis GEO (drivers) · WebSocket/SSE

CAP
───
CP: inventory reservation, order state, payment, shard leadership
AP: availability snapshot, ETA table, catalog, tracking, notifications

STAFF SIGNALS TO HIT
────────────────────
✓ AP-read / CP-write split stated up front (CQRS)
✓ "500 DCs don't need a geo index"; H3 justified by the ETA requirement
✓ Drive time precomputed vs prep/driver time live
✓ Conditional update + CHECK constraint + single-shard txn
✓ Payment-vs-expiry race via CAS; void on lost race
✓ Union availability vs single-DC fulfillment conflict
✓ Physical capacity as the true bottleneck → ETA backpressure
✓ Safety stock / reconciliation for the model-vs-shelf gap
✓ Every read dependency has a degraded mode; writes fail closed
```

---

## 22. Related Guides

- **Proximity Service** — when geohash/quadtree/H3 indexes *are* needed (millions of points); contrast with 500 DCs here
- **Yelp (local business search)** — geo search + text search combination patterns
- **Apache Kafka** — partitioning by key, `acks` + `min.insync.replicas` durability contract for inventory-changes
- **Sharding Strategies** — why the shard key decides which query becomes a fan-out (here: customer order history)
- **CAP/PACELC Theorem** — per-component positioning
- **Circuit Breaker Pattern** — degraded modes for ES, Redis, payment provider
- **Distributed Unique ID Generation** — Snowflake-style `order_id`
- **Leader Election in Distributed Systems** — Postgres shard failover, fencing tokens/epochs
- **Notification System** — order status push/SMS fan-out
- **Stock Market Status Service** — another domain with a "model vs reality" correctness trap class
- **Web Crawler** — format reference; spider traps as the analogous trap class

---

*Guide compiled for Senior & Staff-level system design interview preparation.*
*Topics: Local Delivery, Geo/ETA, Inventory Consistency, Race Conditions, CQRS, CDC, Sharding, CAP/PACELC.*
