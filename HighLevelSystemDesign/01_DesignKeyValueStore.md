# Distributed Key-Value Store — System Design Interview Guide
### Senior & Staff Engineer Level

---

## Table of Contents

1. [Problem Statement & Scope](#1-problem-statement--scope)
2. [Requirements](#2-requirements)
3. [Capacity Estimation](#3-capacity-estimation)
4. [High-Level Architecture](#4-high-level-architecture)
5. [Component 1: Consistent Hashing & Partitioning](#5-component-1-consistent-hashing--partitioning)
6. [Component 2: Replication](#6-component-2-replication)
7. [Component 3: Quorum Consistency (W / R / N)](#7-component-3-quorum-consistency-w--r--n)
8. [Component 4: Conflict Resolution](#8-component-4-conflict-resolution)
9. [Component 5: Write Path](#9-component-5-write-path)
10. [Component 6: Read Path & Read Repair](#10-component-6-read-path--read-repair)
11. [Component 7: Storage Engine (LSM-Tree)](#11-component-7-storage-engine-lsm-tree)
12. [Component 8: Anti-Entropy (Merkle Trees)](#12-component-8-anti-entropy-merkle-trees)
13. [Component 9: Failure Detection & Membership (Gossip)](#13-component-9-failure-detection--membership-gossip)
14. [Component 10: Hinted Handoff & Sloppy Quorum](#14-component-10-hinted-handoff--sloppy-quorum)
15. [CAP / PACELC Positioning](#15-cap--pacelc-positioning)
16. [Failure Modes & Mitigations](#16-failure-modes--mitigations)
17. [Scalability & Rebalancing](#17-scalability--rebalancing)
18. [Tunable Consistency & Client Considerations](#18-tunable-consistency--client-considerations)
19. [Senior vs Staff Answer Differentiators](#19-senior-vs-staff-answer-differentiators)
20. [Interview Time Allocation](#20-interview-time-allocation)
21. [Quick-Reference Cheatsheet](#21-quick-reference-cheatsheet)

---

## 1. Problem Statement & Scope

A **distributed key-value store** is a horizontally-scalable data store that exposes a simple `get(key)` / `put(key, value)` / `delete(key)` API, distributes data across many nodes, and stays available and low-latency even as nodes fail or the network partitions. This is the design underlying systems like **DynamoDB, Cassandra, Riak, and Voldemort** — the canonical "Dynamo paper" interview question.

### What the interviewer is really testing

- Can you reason precisely about **partitioning and replication** at scale?
- Do you understand **quorum-based consistency** well enough not to conflate it with consensus?
- Can you handle **conflicting concurrent writes** without a single leader?
- Do you make **explicit CAP/PACELC tradeoffs**, not just recite the theorem?
- Can you describe a **storage engine** (LSM-tree) that makes writes cheap at scale?
- Can you proactively surface **failure modes** — partitions, node loss, hot keys, clock skew?

### In Scope

- Simple key-based access (`get`/`put`/`delete`), optional conditional writes, optional TTL
- Multi-node partitioning, N-way replication, tunable consistency
- Single-region primary design, with multi-region as an extension

### Out of Scope (mention, then defer)

- Complex queries, joins, secondary indexes, multi-key transactions — these push you toward a different system (document store, OLTP database) or an added indexing layer on top of the KV store
- A single-leader design (Raft-replicated KV store like etcd) is a **different point in the design space** — worth naming explicitly as the alternative (see [Section 7](#7-component-3-quorum-consistency-w--r--n))

---

## 2. Requirements

### Functional Requirements

| Requirement | Notes |
|---|---|
| `put(key, value)` | Write or overwrite a value |
| `get(key)` | Return current value(s) — may return multiple **sibling** versions on conflict |
| `delete(key)` | Logical delete via **tombstone**, not physical removal (needed for replica convergence) |
| Versioning | Support causality tracking so clients can reason about "did my write happen before this read?" |
| Optional: TTL | Auto-expire keys |
| Optional: conditional write (CAS) | `put_if_version_matches` — useful for optimistic concurrency |

### Non-Functional Requirements

| Requirement | Target |
|---|---|
| Availability | Writes and reads should succeed even during node failure or network partition ("always writable") |
| Latency | p99 < 10ms for single-key reads/writes within a region |
| Scale | 100s of millions to billions of keys; horizontally scalable by adding nodes |
| Durability | No acknowledged write is lost, even across a single node failure |
| Consistency | **Tunable** — not fixed. Strong enough for read-your-writes when the client asks for it; eventual otherwise |
| Symmetry | No special node types in the data path — every node can coordinate, store, and replicate (avoids a single point of failure/bottleneck) |

**Framing decision to state out loud in the interview:** *"I'm going to design this as an AP-leaning, leaderless, Dynamo-style store, and call out explicitly where I'd swap in a CP, single-leader design (like Raft-backed etcd) instead."* This single sentence signals you know the design space has more than one valid point in it.

---

## 3. Capacity Estimation

```
Assumptions:
  Keys:              500 million
  Avg value size:    1 KB  (avg key + metadata overhead: ~200 bytes)
  Replication factor (N): 3
  Read/write mix:    80% reads / 20% writes
  Peak QPS:          250,000 ops/sec

Traffic split:
  Reads:  200,000/sec
  Writes:  50,000/sec

Raw data (single copy):
  500,000,000 × 1.2 KB ≈ 600 GB

Total stored data (with RF=3):
  600 GB × 3 ≈ 1.8 TB across the cluster

Write fan-out (every write touches N=3 replicas, independent of W):
  50,000/sec × 3 ≈ 150,000 write-ops/sec cluster-wide
  → W (1/QUORUM/N) only changes how many of those 3 ACKs the coordinator waits
    for before replying to the client — it does NOT change how many replicas
    physically receive the write. The coordinator always propagates to all N;
    that's what "replication factor 3" means. So fan-out stays 150,000 ops/sec
    whether W=1, W=QUORUM, or W=N — unlike reads, where R controls how many
    replicas are even contacted in the first place.

Read fan-out (depends on consistency level chosen):
  R=1 (fast, eventual):     200,000/sec
  R=QUORUM=2 (read-your-writes): 400,000/sec

Total cluster ops/sec:
  R=1:       150,000 + 200,000 = 350,000 ops/sec
  R=QUORUM:  150,000 + 400,000 = 550,000 ops/sec

Per-node capacity (SSD-backed LSM engine, ~10K ops/sec/node sustained):
  Nodes needed (R=1):      350,000 / 10,000 ≈ 35 nodes
  Nodes needed (R=QUORUM): 550,000 / 10,000 ≈ 55 nodes
  → Add 50% headroom for compaction/rebalancing overhead → ~55–85 nodes

Per-node storage:
  1.8 TB / 55 nodes ≈ 33 GB/node  (comfortably fits on SSD; storage is not the bottleneck here — ops/sec is)

Bloom filter memory (reused from URL-dedup sizing, ~10 bits/key at 1% FP):
  500M keys × 10 bits ≈ 625 MB total (before replication) — split across nodes, a few MB/node
```

**Key insight:** unlike the web crawler (storage-bound), a KV store at this scale is usually **throughput-bound**, not storage-bound. The node count is driven by ops/sec and the chosen consistency level (R), not by raw GB. This is worth saying explicitly — it's the kind of "which resource actually constrains me" reasoning interviewers want to hear. All numbers above are illustrative scaffolding for the conversation, not memorized constants.

---

## 4. High-Level Architecture

```
                          ┌───────────────┐
                          │  Client /     │
                          │  App Server   │
                          └───────┬───────┘
                                  │  get/put/delete(key)
                                  │  (token-aware: client knows the ring)
                     ┌────────────▼─────────────┐
                     │   Any Node = Coordinator  │
                     │   (no special "router"    │
                     │    node — every node can  │
                     │    coordinate a request)  │
                     └────────────┬──────────────┘
                                  │  hash(key) → ring position
                    ┌──────────────▼───────────────┐
                    │   Consistent Hash Ring        │
                    │   (with virtual nodes)        │
                    │   → preference list of N      │
                    │     physical nodes             │
                    └──────────────┬────────────────┘
             ┌─────────────────────┼─────────────────────┐
             │                     │                     │
     ┌───────▼──────┐      ┌───────▼──────┐      ┌───────▼──────┐
     │  Replica A   │      │  Replica B   │      │  Replica C   │
     │  ┌─────────┐ │      │  ┌─────────┐ │      │  ┌─────────┐ │
     │  │  WAL    │ │      │  │  WAL    │ │      │  │  WAL    │ │
     │  ├─────────┤ │      │  ├─────────┤ │      │  ├─────────┤ │
     │  │Memtable │ │      │  │Memtable │ │      │  │Memtable │ │
     │  ├─────────┤ │      │  ├─────────┤ │      │  ├─────────┤ │
     │  │SSTables │ │      │  │SSTables │ │      │  │SSTables │ │
     │  │+ Bloom  │ │      │  │+ Bloom  │ │      │  │+ Bloom  │ │
     │  └─────────┘ │      │  └─────────┘ │      │  └─────────┘ │
     └──────┬───────┘      └──────┬───────┘      └──────┬───────┘
            │                     │                     │
            └──────────── Gossip protocol ───────────────┘
            (membership, failure detection, ring state)

            Background, out of request path:
            • Anti-entropy via Merkle tree comparison between replicas
            • Hinted-handoff replay when a failed node rejoins
            • Compaction of SSTables per node
```

**No leader, no single coordinator role.** Any node can receive a client request and act as coordinator for that request — this symmetry is central to Dynamo-style availability: there's no special node whose failure takes down routing.

---

## 5. Component 1: Consistent Hashing & Partitioning

### The Problem With Naive Hashing

`node = hash(key) % num_nodes` reshuffles almost every key when a node is added or removed — a catastrophic amount of data movement at scale.

### Consistent Hashing

Map both nodes and keys onto a fixed hash ring (e.g., 0 to 2^128−1 via SHA-1/MurmurHash). A key belongs to the first node found walking clockwise from its hash position. Adding/removing a node only reshuffles the keys in its immediate ring neighborhood — **O(K/N)** keys move instead of nearly all of them.

### Virtual Nodes (vnodes)

A raw hash ring with one point per physical node causes **uneven load** — some nodes own disproportionately large ring arcs, and a single node failure dumps its entire range onto exactly one neighbor.

```
Without vnodes:                  With vnodes (e.g. 256 per physical node):

   Node A owns a huge arc           Node A's 256 tokens are scattered
   Node B owns a tiny arc           across the ring, interleaved with
   → uneven load                    every other node's tokens
   → A's failure dumps ALL          → load is statistically even
     its data onto ONE neighbor     → A's failure spreads its data
                                       across MANY nodes' recovery load
```

Virtual nodes give you: even load distribution, faster/parallelized recovery after a node failure (many nodes each take a small slice, not one node taking it all), and easy heterogeneous capacity (give a bigger node more vnodes).

*See the [Sharding Strategies guide](#) for the general treatment of consistent hashing, vnodes, and rebalancing — this section applies that pattern specifically to the replica-placement problem.*

### Preference List

For key `K`, walk clockwise from `hash(K)` and collect the first **N distinct physical nodes** (skipping vnodes that map back to a node already in the list). This ordered list is the **preference list** — the set of nodes responsible for storing replicas of `K`.

---

## 6. Component 2: Replication

Each key is replicated to **N** nodes (typically N=3) drawn from its preference list. Replication serves two purposes:

1. **Durability** — survive the loss of up to N−1 nodes without losing the key
2. **Availability** — serve reads/writes even if some replicas are down

### Synchronous vs Asynchronous Replication

| Mode | Latency | Consistency | Use when |
|---|---|---|---|
| Synchronous (wait for all N) | Highest, tail-latency-sensitive | Strongest | Rarely used in Dynamo-style systems — kills availability |
| Quorum (wait for W of N) | Balanced | Tunable | **The default for this design** |
| Asynchronous (ack after 1, replicate in background) | Lowest | Weakest (risk of loss on coordinator failure) | Only for workloads that explicitly trade durability for latency |

### Multi-Datacenter Replication

Preference lists can be extended to span datacenters: replicate synchronously within the local DC (for latency) and asynchronously cross-DC (for disaster recovery). `LOCAL_QUORUM` (quorum within the local DC only) is the practical default for multi-region deployments — waiting on a cross-ocean quorum for every write is a latency non-starter.

---

## 7. Component 3: Quorum Consistency (W / R / N)

This is the single most important — and most often confused — concept in this design. **Precision here is a strong Staff-level signal.**

### Two Different Things Called "Quorum"

| | **Replication Quorum (Dynamo-style)** | **Consensus Quorum (Raft/Paxos-style)** |
|---|---|---|
| Question it answers | "How many replicas must acknowledge a write for it to be considered durable/visible?" | "How many nodes must agree for a single, totally-ordered decision (who's leader, what's the next log entry) to be committed?" |
| Formula | Any **W** and **R** you choose, with **W + R > N** as the design invariant | Strict majority: **⌊N/2⌋ + 1** |
| What it guarantees | The write-set and read-set are guaranteed to **overlap by at least one node** — so a read will see at least one copy of the latest acknowledged write | A **single agreed-upon total order** of operations — true linearizability |
| Does it prevent conflicting writes? | **No.** Two concurrent writes with W=2 can both succeed on non-overlapping majorities of replicas, producing **siblings** that must be reconciled later (vector clocks, LWW, CRDTs) | **Yes**, by construction — only one write can win the log slot for a given position, because there's a single leader ordering all writes |
| Failure model | Leaderless — any node coordinates any request | Leader-based — writes go through a leader; followers replicate the leader's log |
| Example systems | DynamoDB, Cassandra, Riak, Voldemort | etcd, Raft-backed KV stores (Consul), Zookeeper-backed systems, Spanner (with TrueTime added) |

**Why the conflation happens:** both involve counting acknowledgments past a threshold before "success," so it's tempting to treat them as the same mechanism. They are not. Dynamo's `W + R > N` guarantees **set overlap** (you'll *see* the latest write among the versions returned) but says nothing about *ordering* between two writes that raced each other — that's a separate problem solved by [conflict resolution](#8-component-4-conflict-resolution). Raft's majority guarantees a **single order** exists at all, because writes are serialized through one leader before replication even starts.

**How to say this cleanly in an interview:** *"W+R>N gives me read/write set overlap — I'm guaranteed to see the latest acknowledged write among the versions a quorum read returns. It does not give me linearizability, because two concurrent writes can each reach a disjoint majority of replicas and neither one 'wins' — I have to reconcile that with vector clocks. If I wanted true linearizable ordering I'd reach for a Raft-style majority quorum behind a single leader instead — that's a fundamentally different mechanism, not a bigger version of this one."*

### Choosing N, W, R

```
N = 3 is the standard default (tolerates 1 node failure while keeping W+R>N with small W/R)

Strict quorum:      W + R > N            → overlap guaranteed, tunable strength
Read-heavy tuning:   W = N, R = 1        → fast reads, writes must reach everyone (less available on write)
Write-heavy tuning:  W = 1, R = N        → fast writes, reads must reach everyone (less available on read)
Balanced (default):  W = R = QUORUM = ⌈(N+1)/2⌉  → e.g. N=3 → W=R=2

Availability math (N=3):
  W=2, R=2 → tolerates 1 replica down and still meets both W and R
  W=3, R=1 → write fails if ANY replica is down (no write availability during failure)
  W=1, R=1 → always available, but W+R=2 ≤ N=3 → NO overlap guarantee — pure eventual consistency
```

**Staff-level nuance:** `W + R ≤ N` is not a bug — it's a valid, deliberate configuration for workloads (caches, analytics counters, session data) where you're explicitly trading consistency for maximum availability and minimum latency. Naming this as a *choice* rather than a mistake is exactly the kind of tradeoff-fluency interviewers are listening for.

---

## 8. Component 4: Conflict Resolution

Because writes can succeed on different subsets of replicas (especially under [sloppy quorum](#14-component-10-hinted-handoff--sloppy-quorum)), the *same key* can end up with genuinely concurrent, conflicting versions. A leaderless store must resolve this — there's no single leader to serialize writes for you.

### Option 1: Last-Write-Wins (LWW)

Attach a timestamp to every write; on conflict, the higher timestamp wins, loser is discarded.

- **Pro:** trivial to implement, no client-side complexity
- **Con:** **silently loses data** on true concurrent writes; correctness depends on clock synchronization across nodes (NTP drift can cause a causally-earlier write to "win" if its clock is ahead) — Cassandra uses this by default, and it's the right choice when losing the "loser" write is acceptable (e.g., overwriting a cache value)

### Option 2: Vector Clocks

Track causality explicitly instead of relying on wall-clock time. Each write carries a vector clock: `{node_id: counter}`.

```
Example — Client writes key "cart:42":

  Write 1 (via coordinator A): cart = ["shoes"]         VC: {A:1}
  Write 2 (via coordinator A): cart = ["shoes","socks"]  VC: {A:2}     ← causally AFTER write 1 (A:2 descends A:1)

  Network partition. Two clients write concurrently to DIFFERENT coordinators:
  Write 3 (via coordinator A): cart = ["shoes","socks","hat"]   VC: {A:3}
  Write 4 (via coordinator B): cart = ["shoes","socks","gloves"] VC: {A:2, B:1}

  Neither VC descends the other → CONCURRENT, not causally ordered
  → get("cart:42") returns BOTH as "siblings":
       ["shoes","socks","hat"]    {A:3}
       ["shoes","socks","gloves"] {A:2,B:1}
  → Client (or app logic) merges: ["shoes","socks","hat","gloves"]
  → Resolved write carries a VC descending BOTH parents: {A:4, B:1}
```

- **Pro:** never silently drops a concurrent write — surfaces the conflict instead
- **Con:** pushes reconciliation to the client/application; vector clock size grows with the number of distinct coordinating nodes over a key's history — bounded in practice by pruning the oldest `{node: counter}` entries once the clock exceeds a size threshold

### Option 3: CRDTs (Conflict-free Replicated Data Types)

For specific data shapes — counters, sets, registers — use a data structure whose merge function is **mathematically guaranteed** to converge regardless of write order (commutative, associative, idempotent merge). A G-Counter (grow-only counter) merges by taking the per-node max; an OR-Set merges by union with tombstones for removes.

- **Pro:** conflict resolution happens automatically and correctly, no client involvement
- **Con:** only works for specific data shapes — not a general-purpose value store; adds a small amount of metadata per value

### Decision Table

| Workload | Recommended strategy |
|---|---|
| Cache values, session tokens (losing a write is cheap) | LWW |
| Shopping carts, collaborative documents (losing a write is bad) | Vector clocks + app-level merge |
| Counters, presence sets, tags | CRDTs |
| Strict correctness required (financial ledger) | **Don't use this architecture** — use a CP, single-leader system instead |

---

## 9. Component 5: Write Path

```
1. Client sends PUT(key, value) to a node — a **token-aware client** hashed
   the key locally against its cached ring (Section 5) and sends straight to a
   preference-list node; a **"dumb" client** just sends to any node (e.g. via
   a load balancer), not yet knowing the preference list at all
2. If the receiving node isn't already on the preference list, it hashes the
   key, walks the consistent-hash ring to derive the preference list of N nodes
   (from the cluster's gossiped ring/membership state — not something the
   client invents), and becomes the coordinator, acting on the client's behalf
3. Coordinator generates/increments the vector clock for this write
4. Coordinator sends the write to all N replicas in parallel (including itself if it's one)
5. Each replica:
     a. Appends to its local Write-Ahead Log (durability before ack)
     b. Applies to in-memory memtable
     c. Sends ACK to coordinator
6. Coordinator waits for W acknowledgments (not all N)
7. On W acks received → return SUCCESS to client
8. Remaining (N − W) replicas are updated asynchronously, best-effort
   (if a preference-list node is down, use hinted handoff — see Section 14)
```

**Flow diagram** (N=3, W=2 — coordinator returns as soon as 2 of 3 replicas ack, doesn't wait on the third):

```
 Client              Coordinator                Replica A     Replica B     Replica C
   │                      │                          │             │             │
   │── PUT(key,val) ─────▶│                          │             │             │
   │                      │ hash(key), walk ring →   │             │             │
   │                      │ preference list (if not  │             │             │
   │                      │ already on it)           │             │             │
   │                      │ generate/increment       │             │             │
   │                      │ vector clock             │             │             │
   │                      │                          │             │             │
   │                      │── WRITE(k,v,vc) ────────▶│             │             │
   │                      │── WRITE(k,v,vc) ──────────────────────▶│             │
   │                      │── WRITE(k,v,vc) ────────────────────────────────────▶│
   │                      │                          │WAL→memtable │WAL→memtable │WAL→memtable
   │                      │◀──────── ACK ────────────│             │             │
   │                      │◀──────── ACK ──────────────────────────│             │
   │                      │  W=2 acks reached → return now, don't wait on C      │
   │◀──── SUCCESS ────────│                          │             │             │
   │                      │◀────── ACK (async, best-effort, no client wait) ─────│
```

**Why WAL before memtable ack?** If the node crashes between step 5a and a memtable flush, replaying the WAL on restart recovers the write. Acking before the WAL write would risk silent data loss on crash — this is the same durability pattern as a database redo log or Kafka's own commit log (*see the [Kafka guide](#) for the general write-ahead-log-as-durability-primitive pattern*).

---

## 10. Component 6: Read Path & Read Repair

```
1. Client hashes key → coordinator identifies preference list
2. Coordinator sends GET to R (or all N, take fastest R) replicas in parallel
3. Coordinator waits for R responses
4. Compare vector clocks across the R responses:
     • One version causally dominates all others → return it
     • Multiple concurrent versions exist → return all as siblings (see Section 8)
5. READ REPAIR (background, doesn't block the client response):
     • If any of the R replicas returned a stale version, coordinator pushes
       the latest version to that replica asynchronously
     • This is the cheapest, most frequent form of anti-entropy — it only
       fixes divergence on keys that are actively being read
```

**Flow diagram** (N=3, R=2 — coordinator only needs 2 of 3 responses to answer the client):

```
 Client              Coordinator                Replica A     Replica B     Replica C
   │                      │                          │             │             │
   │── GET(key) ─────────▶│                          │             │             │
   │                      │ hash(key), walk ring →   │             │             │
   │                      │ preference list          │             │             │
   │                      │                          │             │             │
   │                      │── GET(key) ─────────────▶│             │             │
   │                      │── GET(key) ───────────────────────────▶│             │
│                         │    (contact R of N, or all N and take fastest R)     │
   │                      │◀──── value + vclock ─────│             │             │
   │                      │◀──── value + vclock ───────────────────│             │
   │                      │  R=2 responses in → compare vector clocks            │
   │                      │  • one dominates → return it                         │
   │                      │  • concurrent → return siblings (Section 8)          │
   │◀──── value(s) ───────│                          │             │             │
   │                      │──READ REPAIR (async):push latest to stale replica───▶│
   │                      │   (only runs if one of the R responses was stale)    │
```

Read repair keeps *hot* keys converged for free as a side effect of normal traffic. It does **not** fix divergence on cold keys nobody reads — that's what background [Merkle-tree anti-entropy](#12-component-8-anti-entropy-merkle-trees) is for.

---

## 11. Component 7: Storage Engine (LSM-Tree)

Each node needs a local storage engine optimized for the write-heavy, append-friendly pattern this architecture produces. The **Log-Structured Merge Tree (LSM-tree)** is the standard choice (used by Cassandra, RocksDB, LevelDB, HBase).

### Structure

```
Write path (per node):
  Write ──▶ WAL (append-only, durability)
        └─▶ Memtable (in-memory sorted structure, e.g. skip list / red-black tree)

  Memtable fills (e.g. 128–256 MB) ──▶ flush to disk as an immutable SSTable
                                        (Sorted String Table: sorted key→value
                                         pairs, written once, never mutated)

  Over time:
  Disk: [SSTable L0] [SSTable L0] [SSTable L0] [SSTable L1 (bigger, merged)] ...
         ▲ multiple small, unsorted-relative-to-each-other SSTables accumulate

  Background COMPACTION:
    Merge multiple SSTables into fewer, larger, fully-sorted SSTables.
    • Discards overwritten keys (keeps only latest version)
    • Discards tombstones past their grace period (completes deletes)
    • Reduces the number of files a read must check
```

### Read Path Through the Engine

```
get(key):
  1. Check memtable (most recent, in-memory) → if found, return
  2. Check each SSTable, newest to oldest:
       a. Consult that SSTable's BLOOM FILTER first
          → filter says "definitely not present" → skip this SSTable entirely (no disk I/O)
          → filter says "maybe present" → binary search the SSTable's sparse index, then read
       b. First hit (newest SSTable containing the key) wins
```

*(Bloom filter sizing math — bits/key vs. false-positive rate — is worked out in detail in the [Web Crawler guide's dedup section](#); the same structure is reused here to avoid unnecessary disk seeks on reads for keys that don't exist in a given SSTable.)*

### Why LSM Over a B-Tree

| | LSM-Tree | B-Tree |
|---|---|---|
| Writes | Sequential append (memtable + WAL) — very fast, no random I/O | In-place update — random I/O, slower on spinning disks, still costs SSD write amplification |
| Reads | May need to check multiple SSTables (mitigated by Bloom filters + compaction) | Single tree traversal — consistently fast |
| Best for | Write-heavy workloads (this system's profile) | Read-heavy, low-write workloads |
| Cost | Background compaction consumes CPU/IO; read amplification if compaction falls behind | Page splits/fragmentation over time |

**Staff-level note:** naming *why* LSM wins here (write-optimized, sequential I/O, deferred/batched merge cost) rather than just naming "LSM-tree" as a keyword is the differentiator.

### Compaction Strategies

| Strategy | Behavior | Good for |
|---|---|---|
| Size-tiered | Merge SSTables of similar size together | Write-heavy workloads; can cause read amplification (many files to check) |
| Leveled | Organize SSTables into levels with bounded size, non-overlapping key ranges within a level | Read-heavy workloads; more consistent read latency, more compaction I/O |

---

## 12. Component 8: Anti-Entropy (Merkle Trees)

Read repair only fixes divergence on keys someone actually reads. Cold data can silently diverge between replicas after a missed write (e.g., during a partition) and stay diverged forever without a background process to catch it.

### Merkle Tree Comparison

```
Each replica builds a Merkle tree over its key range:

                    root_hash
                   /          \
              H(L+R)          H(L+R)
             /      \        /      \
          H(a,b)  H(c,d)  H(e,f)  H(g,h)
           /  \    /  \    /  \    /  \
          a    b  c    d  e    f  g    h    ← leaves: hash of a key range's contents

Comparison between two replicas:
  1. Exchange root hashes → different? → recurse into children
  2. At each level, only recurse into subtrees whose hashes differ
  3. Leaves that differ identify the exact key ranges that have diverged
  4. Stream only THOSE key ranges between replicas to reconcile

Cost: O(log n) hash comparisons to locate divergence,
      instead of O(n) — comparing every single key between replicas.
```

This runs as a low-priority background process (often scheduled during off-peak hours, rate-limited to avoid competing with foreground traffic) and is the safety net that guarantees **eventual consistency** even for data nobody happens to read.

---

## 13. Component 9: Failure Detection & Membership (Gossip)

With no leader and no central coordinator, nodes need a decentralized way to learn the cluster's membership and detect failures.

### Gossip Protocol

```
Every T seconds, each node:
  1. Picks 1–3 random peers
  2. Exchanges membership state: {node_id, status, heartbeat_counter, version}
  3. Merges received state with local state (keep the higher heartbeat_counter per node)

Convergence: membership changes propagate to the whole cluster in O(log N) rounds
             — same epidemic-spread mathematics as gossip in other distributed systems.
```

### Phi Accrual Failure Detector

A **fixed timeout** ("no heartbeat for 5s = dead") produces false positives under variable network latency and false negatives when it's too lenient. The **Phi Accrual** detector instead computes a continuous suspicion score from the *historical distribution* of heartbeat inter-arrival times:

```
φ = -log10(P(time_since_last_heartbeat | historical_distribution))

Low φ  → heartbeat is on-time relative to history → node considered alive
High φ → heartbeat is late relative to what's normal for THIS node/link → suspect

Each application sets its own φ threshold based on how aggressively
it wants to declare failure vs. tolerate a slow-but-alive node.
```

This adapts per-node/per-link automatically — a node with historically jittery heartbeats needs a longer absence before being suspected than one with rock-steady heartbeats.

---

## 14. Component 10: Hinted Handoff & Sloppy Quorum

### The Problem

Strict quorum (`W + R > N`, always writing to the exact N preference-list nodes) means a write **fails** if fewer than W of those specific N nodes are reachable — even if the rest of the cluster is perfectly healthy. That's a big availability hit for a transient single-node failure.

### Sloppy Quorum + Hinted Handoff

```
Preference list for key K: [Node A, Node B, Node C]
Node B is down.

Sloppy quorum: instead of failing the write, send it to the next healthy
node on the ring past C — say Node D — with a HINT attached:
    hint = "this write actually belongs to Node B; deliver it once B is back"

Node D stores the write in a separate hinted-handoff buffer (not its normal
data path) and continues gossiping to detect B's recovery.

When B rejoins:
    D detects B is alive again (via gossip) → replays all hinted writes
    to B → B applies them → D discards the hints for B.
```

### The Tradeoff (Staff-level nuance)

Sloppy quorum trades **strict quorum overlap** for **write availability**. During the handoff window, a strict-quorum read (`R` nodes from the *original* preference list) might not see the hinted write at all, because it physically lives on Node D, not on any of {A, B, C}. `W + R > N` stops being a hard guarantee the moment sloppy quorum kicks in — it degrades gracefully to eventual consistency for the affected key until the hint is replayed and anti-entropy (Merkle tree sync) catches up any stragglers.

**How to phrase this in an interview:** *"Hinted handoff is what makes this system 'always writable,' but I should be explicit that it's a deliberate, temporary relaxation of the quorum overlap guarantee — not a bug. A strict-quorum design would refuse the write instead and sacrifice availability for that guarantee. Dynamo-style systems choose availability here by default, and that choice is exactly what makes them AP under CAP."*

---

## 15. CAP / PACELC Positioning

*(Cross-reference: [CAP / PACELC Theorem guide](#) for the general framework — this section applies it per-component, as required at Staff level.)*

| Component | CAP Choice | PACELC (no-partition behavior) | Reasoning |
|---|---|---|---|
| Core read/write path (default W/R) | **AP** | **EL** — favors low Latency over strong consistency even absent a partition | Sloppy quorum + async replication to the remaining replicas trades consistency for availability/latency by design |
| Core read/write path (W=N, R=N tuning) | Leans **CP** during partition (writes/reads fail if any replica unreachable) | **EC** — favors Consistency, accepting higher latency | A valid *opt-in* tuning for specific keys/workloads that need it |
| Vector clocks / conflict resolution | **AP**-enabling mechanism | — | Exists specifically to let concurrent writes succeed instead of blocking one of them |
| Storage engine (LSM/WAL) per node | N/A (single-node durability, not distributed consistency) | — | WAL ensures no *acknowledged* write is lost even though the CAP tradeoff is decided above this layer |
| Anti-entropy (Merkle trees) | Supports **AP** | — | The mechanism that makes "eventual" in eventual consistency actually happen |
| Gossip / failure detection | **AP** | — | Membership view itself is eventually consistent; a stale view just means a slightly suboptimal coordinator choice, not incorrect data |
| Hinted handoff | **AP** | — | Explicitly prioritizes write availability over quorum overlap during the handoff window |
| Alternative: Raft/Paxos-backed single-leader KV store | **CP** | **EC** | Named as the explicit alternative design point — sacrifices availability during partition/leader election for linearizability |

**The one sentence that ties it together:** *"Even with zero network partition, this system chooses lower Latency over stronger Consistency by default (the 'ELC' branch of PACELC) — that's a design decision independent of CAP, and it's tunable per-request via W/R, not fixed system-wide."*

---

## 16. Failure Modes & Mitigations

| Failure | Symptom | Detection | Mitigation |
|---|---|---|---|
| Single node failure | Preference-list write/read misses one replica | Gossip marks node suspect (Phi Accrual) | Sloppy quorum + hinted handoff keeps writes flowing; reads still satisfy R from remaining healthy replicas |
| Network partition | Cluster splits into two groups, each thinks the other is down | Gossip convergence stalls across the partition | Each side keeps accepting writes (AP choice) → conflicting versions accumulate → reconciled via vector clocks + anti-entropy once partition heals |
| Concurrent writes to same key | Divergent values on different replicas ("siblings") | Vector clock comparison at read time shows non-descending clocks | App-level merge, LWW, or CRDT merge depending on data shape (Section 8) |
| Clock skew across nodes | If using LWW: a causally-earlier write "wins" because its node's clock is ahead | Hard to detect directly; shows up as "lost" writes reported by users | NTP on all nodes; prefer vector clocks or hybrid logical clocks over raw wall-clock LWW for correctness-sensitive data |
| Hot key / hot partition | One key (celebrity item, viral post) receives disproportionate traffic, overloading its 3 replica nodes | Per-key/per-node throughput metrics spike | Cache the hot key in front of the store; for writable hot keys, consider key-splitting (`key#0..key#9`) with client-side fan-in on read |
| Bloom filter false positives | Occasional unnecessary disk read on a miss | Elevated read latency on specific SSTables | Not a correctness issue — tune bits/key vs. FP-rate tradeoff; monitor and re-size if FP rate creeps up |
| Compaction falling behind | Read amplification rises (many SSTables to check per read); read latency climbs | SSTable count per node / read latency percentiles | Throttle foreground write rate, prioritize compaction I/O, add nodes to reduce per-node write pressure |
| Cascading failure from client retries | A slow/struggling node gets hammered by retrying clients, worsening its overload | Error rate + latency spike correlated with retry volume | Client-side circuit breaker + exponential backoff with jitter (*see the [Circuit Breaker guide](#)*) |
| Sloppy-quorum write "lost" hints | Node holding a hint crashes before replaying it | Hint count/age metrics per node | Replicate hints themselves with a small replication factor, or bound hint TTL and rely on anti-entropy as the final backstop |
| Gossip convergence lag | Coordinator routes to a node it doesn't yet know is down | Elevated timeout rate on writes to a specific node | Speculative retry to the next preference-list node after a short timeout budget |
| Merkle tree sync overload | Anti-entropy competes with foreground traffic, degrading latency | p99 latency spikes during scheduled sync windows | Rate-limit anti-entropy bandwidth/CPU; schedule during low-traffic windows; prioritize by staleness |
| Rebalancing after node add/remove | Data movement causes temporary load spike on nodes streaming data | Elevated CPU/network on affected nodes | Vnodes bound the fraction of ring each physical node owns → bound the blast radius of any single rebalance |

---

## 17. Scalability & Rebalancing

### Adding a Node

```
1. New node is assigned a set of vnode tokens on the ring
   (either evenly split from existing nodes, or manually assigned for
   heterogeneous hardware)
2. New node streams the relevant key ranges from the nodes that
   previously owned those ring positions
3. Once streaming completes, new node joins the preference lists for
   those ranges going forward
4. Old nodes can garbage-collect data they no longer own
```

Because of vnodes, this data movement is spread across **many** existing nodes rather than dumping the entire burden onto one or two neighbors — bounding the performance impact of scaling events.

### Removing a Node (planned or failure)

```
Planned removal: node streams its data OUT to the nodes taking over its
                  vnode ranges before leaving — no availability gap.
Unplanned failure: remaining N-1 replicas + hinted handoff cover writes;
                    once node is confirmed dead (not just slow), its vnode
                    ranges are reassigned and re-replicated to restore N.
```

### Horizontal Scaling Summary

| Component | Scaling strategy |
|---|---|
| Data nodes | Add nodes; vnodes rebalance automatically; bound per-node blast radius |
| Coordinator role | Not a separate tier — scales with data nodes since any node can coordinate |
| Gossip | O(log N) convergence scales gracefully into the thousands of nodes |
| Anti-entropy | Rate-limited and incremental; scales by adding more nodes to share the Merkle-comparison work |
| Client routing | Token-aware clients cache ring topology to route directly to a preference-list node, avoiding an extra coordinator hop |

*(Cross-reference: [Sharding Strategies guide](#) for the general consistent-hashing/vnode rebalancing mechanics applied here.)*

---

## 18. Tunable Consistency & Client Considerations

### Per-Request Consistency Levels

Expose W and R as **per-request** knobs, not fixed system-wide settings:

| Level | Meaning | Use case |
|---|---|---|
| `ONE` | Wait for 1 replica | Fastest; analytics writes, best-effort telemetry |
| `QUORUM` | Wait for ⌈(N+1)/2⌉ | Default balanced choice |
| `ALL` | Wait for all N | Strongest, least available; rarely used |
| `LOCAL_QUORUM` | Quorum within local DC only | Multi-region deployments avoiding cross-DC write latency |

### Read-Your-Writes

Even with quorum reads, a client can fail to see its own recent write if the read happens to hit a replica outside the write's acknowledgment set. Practical fixes:

- **Sticky sessions** — route a client's reads to the same coordinator/replica it wrote through
- **Client-supplied version token** — client remembers the vector clock/version from its write and passes it on the next read; coordinator ensures the returned version is at least that recent (blocking briefly or retrying against another replica if not)

### Monotonic Reads

Without care, a client could read a newer version, then on a subsequent read hit a replica that hasn't caught up yet and see an *older* version — a confusing regression. Fix: route a given client's reads consistently to the same replica set, or track the highest version seen client-side and reject/retry reads that would regress it.

---

## 19. Senior vs Staff Answer Differentiators

### Senior-level expectations

- Correctly describe consistent hashing and why it's used over `hash % N`
- Explain replication factor N and basic quorum reads/writes
- Mention eventual consistency as a concept
- Identify the storage engine as needing to be fast for writes (LSM-tree by name)
- Handle the obvious failure mode: a node goes down, replicas cover it

### Staff-level expectations (additional)

| Topic | What Staff-level adds |
|---|---|
| Quorum | **Explicitly distinguishes** Dynamo-style replication quorum (`W+R>N`, overlap guarantee) from Raft/Paxos consensus quorum (majority, total order) — and states which one this design uses and why |
| Conflict resolution | Names vector clocks *and* explains why W+R>N alone doesn't prevent siblings; discusses LWW/CRDT tradeoffs by workload, not just "use vector clocks" |
| Storage engine | Explains *why* LSM beats B-tree here (sequential write I/O vs. random), not just naming it; covers compaction strategy tradeoffs |
| Anti-entropy | Distinguishes read-repair (reactive, hot-key coverage) from Merkle-tree sync (proactive, cold-key coverage) — most Senior answers only mention one |
| Failure detection | Names Phi Accrual over fixed timeouts, and explains *why* (adapts to per-link jitter) |
| Hinted handoff | Explicitly names the tradeoff: it breaks the strict `W+R>N` overlap guarantee during the handoff window — frames it as a deliberate, temporary relaxation, not hand-waved as "just works" |
| CAP/PACELC | Positions *every* component individually, and adds the PACELC "no-partition" latency/consistency tradeoff on top of CAP, not just CAP alone |
| Tunability | Frames consistency (W/R) as a **per-request** dial, not a system-wide fixed setting |
| Failure modes | Proactively raises hot keys, clock skew's effect on LWW specifically, cascading retry failure, and hint-loss — not just "node down" |

### The single most important Staff differentiator

**Precision about what a guarantee actually guarantees.** Saying "we use quorum for consistency" is Senior-level. Saying *"W+R>N guarantees read/write set overlap, not linearizable ordering — concurrent writes can still race each other onto disjoint majorities, which is why I need a separate conflict-resolution mechanism on top"* is Staff-level. The interviewer is listening for whether you understand the boundary of each mechanism's guarantee, not just its name.

---

## 20. Interview Time Allocation

For a 45-minute system design session:

| Phase | Time | Focus |
|---|---|---|
| Requirements & scope | 4 min | Confirm functional + non-functional; state the AP/leaderless framing decision up front |
| Capacity estimation | 4 min | QPS, storage, node count — flag whether throughput or storage is the binding constraint |
| High-level architecture | 4 min | Draw the ring, replicas, gossip, no-special-node symmetry |
| Partitioning & replication | 6 min | Consistent hashing, vnodes, preference list |
| **Quorum (W/R/N)** | 8 min | The core of the interview — precise treatment of quorum overlap vs. consensus, tunability |
| Conflict resolution | 6 min | Vector clocks with a concrete example; LWW/CRDT tradeoffs |
| Storage engine | 5 min | LSM-tree, WAL, compaction, Bloom filters |
| Anti-entropy & failure detection | 4 min | Merkle trees, read repair, gossip, Phi Accrual |
| Failure modes | 4 min | Hot keys, clock skew, hinted handoff tradeoff, cascading retries |

**What to cut if short on time:** Multi-DC replication detail and CRDT internals. **Never cut:** the quorum precision section — it's the section most likely to separate Senior from Staff signal in the interviewer's notes.

---

## 21. Quick-Reference Cheatsheet

```
KEY ALGORITHMS
──────────────
Partitioning:      Consistent hashing + virtual nodes  |  O(K/N) movement on node change
Quorum:            W + R > N  →  read/write set overlap (NOT linearizability)
Consensus (alt.):  Majority quorum ⌊N/2⌋+1  →  total order (Raft/Paxos) — different mechanism
Conflict handling: Vector clocks (causality)  |  LWW (timestamp)  |  CRDTs (mergeable types)
Storage engine:    LSM-tree  |  WAL → Memtable → SSTable → Compaction
Read optimization: Bloom filter per SSTable  |  ~10 bits/key at 1% FP
Anti-entropy:      Merkle tree comparison  |  O(log n) divergence detection
Failure detection: Phi Accrual  |  adaptive, per-link, avoids fixed-timeout false positives

KEY NUMBERS (illustrative — recompute live in the interview)
───────────
N = 3 replicas (standard default)
W = R = 2 (balanced quorum for N=3)
~10K ops/sec per node (SSD-backed LSM engine, order-of-magnitude)
~10 bits/key for 1% Bloom filter false-positive rate
Memtable flush threshold: ~128–256 MB (implementation-dependent)

KEY SYSTEMS
───────────
Storage engine:     LSM-tree (RocksDB/LevelDB-style) — WAL + memtable + SSTables
Membership:         Gossip protocol + Phi Accrual failure detector
Partitioning:        Consistent hash ring with virtual nodes
Conflict tracking:   Vector clocks (or LWW / CRDTs depending on data shape)
Anti-entropy:        Merkle trees, background, rate-limited
Availability escape: Sloppy quorum + hinted handoff

CAP / PACELC DEFAULT POSITION
──────────────────────────────
AP under partition (sloppy quorum keeps writes flowing)
EL absent partition (favors Latency over Consistency by default, tunable via W/R)
Alternative CP/EC design point: single-leader, Raft/Paxos-backed store

THE QUORUM DISTINCTION (say this explicitly)
───────────────────────────────────────────
W+R>N        → guarantees OVERLAP between write-set and read-set
               → does NOT prevent concurrent writes from conflicting
Majority     → guarantees a single TOTAL ORDER via one leader
               → different mechanism, different guarantee, different failure model

STAFF-LEVEL SIGNALS TO HIT
───────────────────────────
✓ Explicitly separate replication quorum from consensus quorum
✓ Name W+R>N as an overlap guarantee, not a consistency guarantee by itself
✓ Vector clock walkthrough with a concrete sibling example
✓ LSM-tree chosen and justified by write pattern, not just named
✓ Read repair (reactive) vs. Merkle anti-entropy (proactive) — both, distinguished
✓ Phi Accrual over fixed timeouts, with the "why"
✓ Hinted handoff framed as a deliberate, temporary quorum relaxation
✓ Per-component AND PACELC (not just CAP) positioning
✓ Consistency level (W/R) framed as tunable per-request, not fixed
✓ Proactive failure modes: hot keys, clock skew + LWW interaction, cascading retries
```

---

*Guide compiled for Senior & Staff-level system design interview preparation.*
*Topics: Distributed Key-Value Store, Consistent Hashing, Quorum Consistency, Vector Clocks, LSM-Trees, CAP/PACELC Theorem.*
*Cross-references: Sharding Strategies, CAP/PACELC Theorem, Apache Kafka, Circuit Breaker Pattern, Web Crawler (Bloom filter sizing).*