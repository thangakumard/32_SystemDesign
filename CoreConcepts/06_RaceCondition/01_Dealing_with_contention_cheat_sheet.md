# Dealing with Contention — System Design Cheat Sheet

> **Source:** [Hello Interview — Dealing with Contention](https://www.hellointerview.com/learn/courses/system-design/lesson/contention/dealing-with-contention)

---

## Table of Contents
1. [Overview & Core Problem](#1-overview--core-problem)
2. [When Does Contention Occur?](#2-when-does-contention-occur)
3. [Strategies & Techniques](#3-strategies--techniques)
   - [Single-Node / Database-Level Solutions](#single-node--database-level-solutions)
     - [Pessimistic Locking](#1-pessimistic-locking)
     - [Optimistic Concurrency Control (OCC)](#2-optimistic-concurrency-control-occ)
     - [Atomic Operations & Database Updates](#3-atomic-operations--database-updates)
   - [Distributed / High-Scale Solutions](#distributed--high-scale-solutions)
     - [Distributed Locking](#4-distributed-locking)
     - [Message Queues & Serialized Processing](#5-message-queues--serialized-processing)
     - [Partitioning & Sharding (Hotspot Mitigation)](#6-partitioning--sharding-hotspot-mitigation)
     - [Reservations & Time-Bound Leases](#7-reservations--time-bound-leases)
     - [Batching & Wave Processing](#8-batching--wave-processing)
4. [Trade-offs Comparison Matrix](#4-trade-offs-comparison-matrix)
5. [Common Interview Use Cases & Applications](#5-common-interview-use-cases--applications)
6. [Interview Deep Dives & Red Flags](#6-interview-deep-dives--red-flags)

---

## 1. Overview & Core Problem

**Contention** occurs when multiple concurrent processes or users compete for the exact same resource at the exact same time (e.g., purchasing the last ticket for a concert, buying a flash sale item, updating a hot counter, or bidding in an auction).

### The Race Condition Problem
Consider a simple read-modify-write pattern on a database column:
```sql
-- Step 1: Read current inventory
SELECT available_seats FROM concerts WHERE concert_id = 'weeknd_tour';

-- Step 2: Application checks if available_seats > 0...

-- Step 3: Decrement seat count
UPDATE concerts SET available_seats = available_seats - 1 WHERE concert_id = 'weeknd_tour';
```

If two users (Alice and Bob) run Step 1 simultaneously when `available_seats = 1`, both read `1`, both pass the application check, and both run Step 3. As a result:
* **Outcome:** Overselling / double-booking / corrupted data state.
* **Goal:** Ensure correctness (consistency & atomicity) without destroying system throughput and availability.

---

## 2. When Does Contention Occur?

Typical high-contention interview scenarios:
* **Flash Sales / Limited Drops:** E-commerce items with limited stock (e.g., Ticketmaster, sneaker drops).
* **Distributed Counters / Aggregations:** Likes, view counts, ad click aggregators, live comments.
* **Auction & Bidding Systems:** Rapid concurrent price increases on a single item.
* **Seat / Resource Reservations:** Booking flights, hotel rooms, or ride-sharing matches (Uber drivers).

---

## 3. Strategies & Techniques

### Single-Node / Database-Level Solutions

#### 1. Pessimistic Locking
* **Mechanism:** Lock the database row/resource *before* reading or updating. Prevents any other process from reading or updating until the current transaction finishes.
* **Implementation (SQL):**
  ```sql
  BEGIN TRANSACTION;
  SELECT available_seats FROM concerts WHERE concert_id = 'weeknd_tour' FOR UPDATE;
  -- Application checks seats > 0
  UPDATE concerts SET available_seats = available_seats - 1 WHERE concert_id = 'weeknd_tour';
  COMMIT;
  ```
* **Pros:** Guarantees strict correctness; simple to reason about.
* **Cons:** High latency under heavy contention; risk of **deadlocks**; holds open database connections.

#### 2. Optimistic Concurrency Control (OCC)
* **Mechanism:** Assume conflicts are rare. Allow reads without locking, but include a version number or timestamp when writing. If the version changed in the background, abort and retry.
* **Implementation (SQL):**
  ```sql
  -- Step 1: Read value and version
  SELECT available_seats, version FROM concerts WHERE concert_id = 'weeknd_tour';

  -- Step 2: Update only if version hasn't changed
  UPDATE concerts 
  SET available_seats = available_seats - 1, version = version + 1 
  WHERE concert_id = 'weeknd_tour' AND version = @read_version;
  ```
* **Pros:** Non-blocking reads; great performance under low-to-moderate contention.
* **Cons:** High retry rate and wasted CPU cycles under extreme contention; prone to the **ABA problem** if not properly versioned.

#### 3. Atomic Operations & Database Updates
* **Mechanism:** Push the conditional logic directly into the database engine in a single, atomic update operation without needing explicit locks or two-step queries.
* **Implementation (SQL / DynamoDB):**
  ```sql
  UPDATE concerts 
  SET available_seats = available_seats - 1 
  WHERE concert_id = 'weeknd_tour' AND available_seats > 0;
  ```
* **Pros:** Extremely fast; no application-level lock needed; atomic execution inside the storage engine.
* **Cons:** Limited to single-row updates and simple logic.

---

### Distributed / High-Scale Solutions

#### 4. Distributed Locking
* **Mechanism:** Use a centralized, fault-tolerant state store (Redis with Redlock, ZooKeeper, or Etcd) to acquire a mutual exclusion lock across microservices before modifying data.
* **Implementation:**
  * Client requests lock: `SET lock:item_123 client_id NX PX 5000` (5-second TTL).
  * Performs operation on DB/Service.
  * Releases lock via Lua script (verifying `client_id` to prevent deleting someone else's lock).
* **Pros:** Works across distributed microservices and heterogeneous databases.
* **Cons:** Adds network overhead; lease TTL management is difficult (fencing tokens required to prevent split-brain issues when garbage collection pauses happen).

#### 5. Message Queues & Serialized Processing
* **Mechanism:** Convert concurrent writes into a sequential queue. Channel all requests through a queue partition (e.g., Kafka partition key = `item_id`), forcing a single worker to process updates strictly sequentially.
* **Pros:** Completely eliminates race conditions; turns high-concurrency spikes into smooth, predictable throughput.
* **Cons:** Increases latency for user response (asynchronous flow required); requires job status polling/websockets for client notification.

#### 6. Partitioning & Sharding (Hotspot Mitigation)
* **Mechanism:** Instead of having 1 global counter or single row, split the resource into $N$ buckets (e.g., 10 distributed counter slots). Writes are randomly routed to any bucket; total count is calculated on read via aggregation.
* **Pros:** Linearly scales write throughput for counters and aggregations.
* **Cons:** Adds complexity to reads (must aggregate across all buckets); hard to apply to non-splittable distinct inventory items (e.g., a specific seat).

#### 7. Reservations & Time-Bound Leases
* **Mechanism:** Decouple "reservation" from "payment/final settlement." Grant temporary holding rights (e.g., 10-minute hold on a ticket) using Redis or ephemeral database records. If not completed within TTL, release back to pool.
* **Pros:** Reduces lock duration on primary database; improves UX by guaranteeing the item to the buyer while checking out.
* **Cons:** Requires background garbage collection / TTL cleanup to reclaim expired holds.

#### 8. Batching & Wave Processing
* **Mechanism:** Do not process requests individually in real time. Accumulate bids, requests, or updates over a fixed time window (e.g., 100ms or 1-second waves), and select winners or process updates in a single batch script.
* **Pros:** Massively reduces database IOPS and locking contention.
* **Cons:** Introduces minor artificial latency.

---

## 4. Trade-offs Comparison Matrix

| Approach | Consistency | Throughput / Scalability | Latency | Implementation Complexity | Best Used For |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Pessimistic Locking** | Strongest | Low | High | Low | Low-concurrency, high-stakes transactional operations |
| **Optimistic Locking (OCC)** | Strong | Medium-High | Low (low load) / High (high load) | Low-Medium | Read-heavy workloads with occasional concurrent writes |
| **Atomic DB Updates** | Strong | High | Very Low | Low | Single-row inventory/counter decrements |
| **Distributed Locks** | Strong | Medium | Medium | Medium-High | Microservice coordination across disparate databases |
| **Queue-based Serialization**| Strong | Very High (Buffered) | Medium-High | Medium | Extreme write spikes (Flash sales, Ticketmaster) |
| **Sharded Counters** | Eventual / Strong | Extremely High | Low | Medium | High-throughput counters (Likes, view counts) |
| **Reservation Leases** | Strong | High | Low | Medium | Multi-step checkout flows (flight/hotel seats) |

---

## 5. Common Interview Use Cases & Applications

1. **Ticketmaster / Concert Tickets:**
   * *Approach:* Redis-based time-bound reservation leases combined with Queue-based ordering per event.
2. **Flash Sale Inventory (E-commerce):**
   * *Approach:* In-memory pre-decrement in Redis (atomic `DECR`) $\rightarrow$ Async queue worker $\rightarrow$ MySQL background sync.
3. **Distributed Counter (Likes / Views):**
   * *Approach:* Counter sharding / bucketing in DB, or Redis buffer aggregated periodically via Kafka + Flink.
4. **Live Bidding / Auction Service:**
   * *Approach:* In-memory stateful actor (or Redis) maintaining highest bid per item with single-threaded updates, persisting periodically to DB.

---

## 6. Interview Deep Dives & Red Flags

### Common Interviewer Probe Questions
* **"How do you handle deadlocks with pessimistic locking?"**
  * *Answer:* Always acquire locks in a globally deterministic order (e.g., sort IDs alphabetically before acquiring locks) and enforce strict lock timeouts.
* **"What is the ABA problem in Optimistic Concurrency?"**
  * *Answer:* If a value changes from A to B and back to A, a naive comparison thinks nothing changed. Solve this by using monotonically increasing **version numbers** or UUID timestamps instead of raw value checks.
* **"What happens if a worker holding a distributed lock crashes?"**
  * *Answer:* Use lock TTLs combined with auto-renewal heartbeats (e.g., Redisson watchdog) and issue **fencing tokens** (incrementing sequence numbers) so storage layers reject late requests from stalled workers.

### Red Flags to Avoid
* ❌ **Relying on application-level read-then-write without database locks or atomic updates.**
* ❌ **Using heavy distributed transactions (2PC / Two-Phase Commit) for high-throughput public APIs.**
* ❌ **Forgetting about lock expiration and TTLs when using Redis or ZooKeeper for distributed locking.**
* ❌ **Over-engineering with distributed locks when a simple SQL atomic update (`WHERE count > 0`) is sufficient.**