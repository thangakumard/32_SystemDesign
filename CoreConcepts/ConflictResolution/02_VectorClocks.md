# Vector Clocks & Conflict Resolution in Quorum Systems

## The core problem

In a quorum system with parameters **(N, R, W)** — N replicas, R nodes contacted per read, W nodes contacted per write — you don't require all N replicas to agree on every write. You only require W acknowledgments. That's what gives you availability during partitions, but it also means two clients can write to *different* subsets of replicas at nearly the same time, each starting from the same stale prior version.

When a later read touches a mix of replicas (R of them), it may see two versions that both claim to descend from the same parent. The system needs a way to tell:

- One version **causally follows** the other → safe to discard the older one.
- The versions are **concurrent** → neither write knew about the other, and both need to be preserved and reconciled.

Vector clocks are the mechanism for making that distinction without relying on synchronized wall clocks.

## How the vector clock works

Each version of an object carries a clock: a map of `{replica_id: counter}`. Every time a coordinator node accepts a write, it increments its own counter in that object's clock.

```
D1: [Sx:1]        first write, coordinated by node Sx
D2: [Sx:2]        client read D1, wrote again, still via Sx
```

### Comparing two clocks A and B

| Relationship | Condition | Meaning |
|---|---|---|
| A descends from B | Every counter in A ≥ corresponding counter in B, and at least one is strictly greater | B is stale — discard it |
| A and B are concurrent | Some counters in A are higher, some in B are higher | Conflict — neither write saw the other |
| A equals B | All counters match | Same version |

## Worked example: branch and merge

```
                    D1: [Sx:1]
                 "cart created"
                        |
                        v
                    D2: [Sx:2]
                "+ milk (via Sx)"
                    /        \
                   v          v
     D3: [Sx:2, Sy:1]      D4: [Sx:2, Sz:1]
     "+ flour (via Sy)"    "+ eggs (via Sz)"
                   \          /
                    v        v
            D5: [Sx:3, Sy:1, Sz:1]
              "reconciled cart"
```

- **D3** and **D4** both descend from D2, but neither has seen the other's increment (`Sy:1` vs `Sz:1` — neither dominates). That's the signature of a write that happened on two different coordinators before either replicated to the other.
- **D5** is the reconciled version: its clock is the component-wise max of D3 and D4, plus a fresh increment. It causally dominates both parents, so future comparisons resolve cleanly again.

## Resolving the conflict

When a client reads with quorum R and the contacted replicas return divergent clocks, the storage layer can't pick a winner on its own — that would silently drop data. Three things typically happen:

1. **Return all siblings to the client/application.** This is Dynamo's original approach. The app has semantic knowledge (e.g., a shopping cart can be merged by unioning items) that the storage engine doesn't.
2. **Client writes back the merged result** with a new clock that's the component-wise max of the siblings plus its own increment (`[Sx:3, Sy:1, Sz:1]` above).
3. **Read repair** propagates the reconciled version to the replicas that were still holding stale data, so the quorum converges without waiting for the next natural write.

## Where quorum parameters matter — and where they don't

- **`R + W > N`** guarantees any read quorum and write quorum overlap by at least one node. This is what guarantees you'll *detect* a conflict at read time — you're guaranteed to see at least one copy of any successfully written version.
- It does **not** prevent the conflict from occurring in the first place. Two coordinators can each collect W acks from disjoint-enough replica sets during a partition (the "sloppy quorum" case with hinted handoff), and both writes succeed independently.
- **Takeaway:** `R + W > N` controls *visibility*, not *ordering*. Vector clocks exist precisely to handle ordering once visibility has surfaced a conflict.

## Tradeoffs vs. the alternatives

| Approach | Detects conflicts? | Needs synchronized clocks? | Cost |
|---|---|---|---|
| **Last-Writer-Wins** (wall-clock timestamp) | No — silently drops one write | Yes (clock skew causes real data loss) | Cheap, O(1) per version |
| **Vector clocks** | Yes, precisely | No | Clock size grows with number of distinct writer nodes ("sibling explosion" if unpruned) |
| **Lamport timestamps** | Only total order, not causality | No | Cheaper, but can't distinguish concurrent from ordered |
| **CRDTs** | Conflicts resolve automatically via merge semantics | No | No app-level reconciliation needed, but only works for data types with well-defined merge functions (counters, sets, etc.) |

## Interview-relevant failure mode

Unpruned vector clocks grow unboundedly as more distinct nodes write to an object over time. Real systems (Dynamo, Riak) cap clock size and drop the oldest `(node, counter)` entries, trading a small amount of historical precision for bounded metadata size.

Notably, **Cassandra moved away from vector clocks entirely** and uses LWW with client-supplied timestamps — accepting the data-loss risk in exchange for operational simplicity and lower per-write overhead. This is a good example to cite in a system design interview when discussing the tension between correctness guarantees and operational simplicity.
