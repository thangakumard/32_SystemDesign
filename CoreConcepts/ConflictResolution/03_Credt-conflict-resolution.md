# CRDTs vs. Quorum-Based Conflict Resolution

## 1. Why Conflict Resolution Exists at All

Any system that replicates mutable state across multiple nodes — for availability, latency, or partition tolerance — eventually faces the same problem: **two replicas accept writes independently, then disagree.** Someone (or something) has to decide what the "true" value is once the replicas talk again.

There are two broad philosophies for resolving this:

| Approach | Core idea | When resolution happens |
|---|---|---|
| **CRDTs** | Design the data type so *any* merge order produces the same result — mathematically, not by voting | At merge time, deterministically, no coordination needed |
| **Quorum-based** | Require enough replicas to agree *before* a read or write is considered valid, then reconcile stragglers | At read/write time, via overlapping replica sets |

This doc covers both, then compares them directly — since in practice (Cassandra, DynamoDB, Riak, Cosmos DB) you often see quorum reads/writes *and* CRDT-like merge logic used together.

---

## 2. CRDTs: Conflict-Free Replicated Data Types

A CRDT is a data structure whose **merge function** is mathematically guaranteed to converge, regardless of:
- the order updates arrive in,
- how many times an update is applied,
- or which replica you ask.

This guarantee comes from three algebraic properties the merge operation must satisfy — the same properties that define a **join-semilattice**:

- **Commutative**: `merge(a, b) == merge(b, a)` — order doesn't matter
- **Associative**: `merge(merge(a, b), c) == merge(a, merge(b, c))` — grouping doesn't matter
- **Idempotent**: `merge(a, a) == a` — applying the same update twice is harmless (handles retries/duplicates for free)

If a merge function has all three properties, replicas that have seen the same set of updates — in *any* order, with *any* duplication — will always converge to the same state. This property is called **Strong Eventual Consistency (SEC)**: not just "will eventually agree" (Eventual Consistency), but "will agree the instant they've seen the same updates, no reconciliation step required."

### 2.1 Two Flavors of CRDT

**State-based (CvRDT — Convergent):**
- Each replica keeps its full state.
- On sync, replicas exchange entire state and call `merge(local, remote)`.
- Simple to reason about; more network traffic (whole state, though delta-CRDTs mitigate this by shipping only recent deltas).

**Operation-based (CmRDT — Commutative):**
- Replicas exchange *operations*, not state (e.g., "increment by 1", "add element X").
- Requires a reliable, causal-order-preserving broadcast channel (at-least-once, no reordering of causally related ops).
- Less bandwidth, but a stronger delivery requirement.

### 2.2 Common CRDT Types

**G-Counter (Grow-only Counter)**
Each replica tracks its own increment count in a vector; the value is the sum.
```java
class GCounter {
    Map<String, Long> counts = new HashMap<>(); // replicaId -> count

    void increment(String replicaId) {
        counts.merge(replicaId, 1L, Long::sum);
    }

    long value() {
        return counts.values().stream().mapToLong(Long::longValue).sum();
    }

    GCounter merge(GCounter other) {
        GCounter result = new GCounter();
        Set<String> allKeys = new HashSet<>(counts.keySet());
        allKeys.addAll(other.counts.keySet());
        for (String k : allKeys) {
            result.counts.put(k, Math.max(
                counts.getOrDefault(k, 0L),
                other.counts.getOrDefault(k, 0L)
            ));
        }
        return result;
    }
}
```
Merge is per-key `max` — commutative, associative, idempotent. This is the classic teaching example.

**PN-Counter (Increment/Decrement Counter)**
Two G-Counters internally — one for increments (`P`), one for decrements (`N`). Value = `P.value() - N.value()`.

**G-Set (Grow-only Set)**
Merge = set union. Can add elements, never remove.

**2P-Set (Two-Phase Set)**
An "add set" and a "remove set" (tombstones). An element is present if it's in the add set and *not* in the remove set. Limitation: once removed, an element can never be re-added (the tombstone is permanent).

**OR-Set (Observed-Remove Set)**
Fixes 2P-Set's re-add problem by tagging each `add(element)` with a unique ID. `remove(element)` only removes the specific tags observed at the time of removal. A concurrent `add` with a new tag survives a concurrent `remove`. This is the set CRDT used in practice (e.g., Riak's Set, Akka's `ORSet`).

**LWW-Register (Last-Write-Wins Register)**
Stores `(value, timestamp)`. Merge keeps the one with the higher timestamp (ties broken by replica ID for determinism). Simple, but **can silently drop concurrent writes** — the "loser" write just disappears. Often combined with wall-clock or hybrid logical clocks (HLC) to reduce arbitrary-looking outcomes.

**MV-Register (Multi-Value Register)**
Instead of picking a winner like LWW, keeps *all* concurrently-written values (tagged with vector clocks) and hands the conflict to the application/user to resolve — this is exactly how Amazon's original Dynamo paper handled shopping-cart conflicts.

**RGA / Sequence CRDTs (e.g., used in collaborative text editors)**
Each character/element gets a unique, globally-ordered ID (often `(counter, replicaId)`). Concurrent inserts at the "same" position are ordered deterministically by ID, so two people typing at the same cursor position converge without garbling text. This is the family behind Automerge, Yjs, and Google Docs–style editors.

### 2.3 The Trade-off CRDTs Make

CRDTs push complexity into **data type design** so that *no coordination is needed at write time*. The cost:
- You must pick a merge semantic in advance (LWW loses data silently; OR-Set/MV-Register keep more but grow tombstones/metadata over time).
- Metadata overhead (vector clocks, tags, tombstones) can grow unbounded without periodic garbage collection ("causal stability" tracking).
- Not every data structure has an obvious conflict-free merge (e.g., "the current bank balance" doesn't CRDT-ify cleanly — that's why counters, not raw values, are the CRDT primitive for numeric state).

---

## 3. Quorum-Based Conflict Resolution

The alternative (used by Dynamo, Cassandra, Riak's quorum mode, DynamoDB): don't design the type to auto-merge — instead, control **how many replicas must participate** in each read and write so that reads and writes always overlap by at least one replica.

### 3.1 The N/R/W Model

- **N** = number of replicas holding a piece of data
- **W** = number of replicas that must acknowledge a write before it's considered successful
- **R** = number of replicas that must respond to a read before returning a result

**Quorum condition:** `R + W > N` guarantees every read overlaps with the most recent successful write on at least one replica — so the read is guaranteed to *see* the latest write, even if it also sees stale ones.

Common configuration: `N=3, W=2, R=2` — tolerates one replica being down for both reads and writes, and `R+W=4 > N=3` holds.

### 3.2 Detecting and Resolving Conflicts

Overlap guarantees you'll *see* conflicting versions, but something still has to pick a winner:

- **Timestamps / Last-Write-Wins**: simplest, same data-loss risk as LWW-Register above.
- **Vector clocks**: each write is tagged with a per-replica version vector. Two versions are either causally ordered (one strictly "happened after" the other → keep the newer) or **concurrent** (neither dominates → both are returned to the client as "siblings," as in Dynamo/Riak, and the application resolves them — e.g., merging two shopping carts).
- **Read-repair**: when a quorum read discovers replicas with divergent versions, the coordinator writes the resolved value back to the stale replicas asynchronously, healing the divergence without waiting for a background anti-entropy pass.
- **Hinted handoff**: if a replica was down during a write, another node temporarily holds a "hint" and replays the write once the replica returns — reduces how often conflicts arise in the first place.
- **Anti-entropy / Merkle trees**: background process comparing replica states in tree form to efficiently find and repair divergence without transferring all data.

### 3.3 The Trade-off Quorums Make

Quorums push complexity into **the read/write path** rather than the data type:
- Any data type can be replicated this way — no special merge algebra required.
- But you pay a coordination cost on every operation (waiting for `W` or `R` acknowledgments), and concurrent writes can still produce genuine conflicts that need vector clocks + application-level (or LWW) resolution.
- Partition tolerance is softer: if you can't reach a quorum, the operation fails outright (unless you relax to "sloppy quorums" with hinted handoff, trading consistency for availability).

---

## 4. CRDTs vs. Quorum: Direct Comparison

| Dimension | CRDTs | Quorum-based |
|---|---|---|
| Where conflict is resolved | Merge function (data type design) | Read/write coordination + app logic |
| Coordination needed per op | None — writes are local-first | Yes — must contact `W` or `R` replicas |
| Availability under partition | Full (AP) — replicas accept writes independently | Degraded if quorum unreachable, unless sloppy quorum used |
| Data loss risk | Depends on CRDT choice (LWW: yes; OR-Set/MV-Register: no) | Depends on resolution policy (LWW: yes; vector-clock siblings: no, but pushes work to app) |
| Metadata overhead | Per-element tags/tombstones, can grow over time | Per-write vector clocks, bounded by replica count |
| Best fit | Offline-first apps, collaborative editing, counters/sets/registers with clear merge semantics | General-purpose keystores where tunable consistency (`R`, `W`) matters more than type-specific merges |
| Real-world examples | Redis Enterprise CRDBs, Riak DT, Akka Distributed Data, Automerge, Yjs, Soundcloud's Roshi | Amazon DynamoDB, Apache Cassandra, Riak (core), Voldemort |

**In practice, these aren't mutually exclusive.** Riak, for instance, offers both a quorum-based key-value mode *and* built-in CRDTs (`Set`, `Map`, `Counter`) layered on top — you pick the model per bucket depending on whether the data has a natural conflict-free merge.

---

## 5. Rule of Thumb

- If the data type has an obvious, safe merge (counters, sets, collaborative text, "add to cart") → **CRDT**. You get local-first writes and no coordination cost.
- If the data is arbitrary and you need tunable consistency/availability trade-offs across many different data shapes → **quorum + vector clocks**, and let the application resolve true conflicts when they surface.
- If you need strict single-value correctness (e.g., a bank balance, a unique constraint) → neither is sufficient alone; you need consensus (Raft/Paxos) or a serializable transaction layer on top.
