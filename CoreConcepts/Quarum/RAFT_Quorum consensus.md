# Raft: A Guide to Distributed Consensus

## 1. The Problem Raft Solves

Suppose you need to store a single piece of data — say, today's price of potatoes. The simplest approach is to keep it in a file on a single server. But that server is a single point of failure: if it goes down (hardware failure, a flooded data center, a network partition), the data becomes unavailable or is lost entirely.

The obvious fix is replication: store the value on multiple servers so that if one goes down, others can still serve requests. But replication introduces a harder problem — **consistency**. If one server says the price is 12, another says 23, and a third says 10, which one is correct? Without a way to keep replicas in agreement, redundancy just multiplies your sources of truth instead of protecting the one you have.

**Consensus algorithms** solve this: they let a cluster of nodes agree on a single, consistent sequence of state changes, even when some nodes fail or messages are delayed. Raft is one such algorithm, designed specifically to be easier to understand and implement than its predecessor, Paxos, while providing equivalent guarantees.

## 2. The Core Idea: A Replicated Log

Raft's central abstraction is the **replicated log**. Every node keeps a log — an ordered, append-only list of commands. Each entry represents an operation to perform on the data (set a value, delete a key, increment a counter, etc.).

The key insight: **if every node applies the same sequence of operations, in the same order, they all end up in the same state.** So the problem "keep replica state consistent" reduces to the simpler problem "keep replica logs identical." Raft's entire job is to make sure every node's log converges to the same sequence of entries.

Each log entry has two pieces of metadata beyond the command itself:
- **Index** — its position in the log.
- **Term** — the "election period" during which it was created (explained below).

## 3. Cluster Roles and the Term Concept

At any given time, every node in a Raft cluster is in exactly one of three states:

| Role | Description |
|---|---|
| **Leader** | Handles all client requests, appends entries to its own log, and replicates them to followers. There is at most one leader at a time. |
| **Follower** | Passive. Accepts log entries and heartbeats from the leader; votes in elections. |
| **Candidate** | A transitional state used only during a leader election. |

Time is divided into **terms**, numbered with monotonically increasing integers. Each term begins with an election. If the election succeeds, one node becomes leader for the rest of that term. If no leader emerges (a split vote), the term ends without a leader and a new election starts. Every node stores the latest term it has seen, and terms act as a logical clock — they let nodes detect and reject stale information from an outdated leader.

## 4. Leader Election

The leader keeps the cluster alive by sending periodic **heartbeats** (empty `AppendEntries` RPCs) to every follower. As long as a follower keeps hearing from the leader, it stays passive.

If a follower doesn't hear a heartbeat within its **election timeout**, it assumes the leader has failed. It then:
1. Increments its current term.
2. Transitions to candidate.
3. Votes for itself.
4. Sends `RequestVote` RPCs to every other node.

A node wins the election if it receives votes from a **majority** of the cluster (including its own vote). Once it wins, it immediately becomes leader and starts sending heartbeats to establish authority and suppress further elections.

**Why timeouts are randomized:** if every follower used the same fixed timeout, they would all notice the leader's absence simultaneously, all become candidates simultaneously, and likely split the vote — with no majority reached, no leader elected, and the process repeating indefinitely. Raft avoids this by randomizing each node's election timeout (typically within a range like 150–300ms). This makes it very likely that one node times out first, starts its election, and gathers votes before others even become candidates.

**Election restriction (a safety rule, not just a performance detail):** a node will only grant its vote to a candidate whose log is *at least as up-to-date* as its own. "Up-to-date" is determined by comparing:
1. The term of the last log entry (higher term wins).
2. If terms are tied, the length of the log (longer log wins).

This prevents a node with stale or incomplete data from becoming leader, which is essential — since the leader is the sole source of truth for replication, an out-of-date leader would cause committed data to be lost.

## 5. Log Replication

Once a leader is established, it handles all writes. For each client request:

1. The leader appends the command to its **own** log as a new entry, tagged with the current term and the next index.
2. It sends an `AppendEntries` RPC containing this new entry to all followers in parallel. This RPC also includes the index and term of the entry immediately *preceding* the new one — this is used for consistency checking (see §6).
3. Each follower that successfully appends the entry replies with success.
4. Once the leader hears success from a **majority** of the cluster (itself included), the entry is considered **committed** — it's now safe and durable, guaranteed not to be lost even if nodes subsequently fail.
5. The leader applies the entry to its own state machine and returns the result to the client.
6. On the next heartbeat/`AppendEntries`, the leader informs followers of the new **commit index**, and each follower applies the entry to its own state machine.

Note the important distinction between **appended** and **committed**:
- *Appended* means a follower has written the entry to its local log — but it isn't final yet, because the follower doesn't know whether the leader reached enough other nodes.
- *Committed* means a majority has confirmed the entry, so it's guaranteed to persist even through leader failure.

This majority-acknowledgment requirement is why Raft (like most consensus protocols) tolerates the failure of a minority of nodes: a cluster of `2f + 1` nodes remains available and consistent as long as no more than `f` nodes fail simultaneously.

## 6. Keeping Logs Consistent After a Node Rejoins

Consider a follower that goes offline temporarily. While it's down, the leader continues to commit new entries with the remaining majority. When the node comes back, its log is now behind — it might have entries 1–3 while the leader is already on entry 6.

Raft can't simply append entry 6 onto that stale log; the log would have a gap and no longer represent a valid, ordered history. Instead, the **Log Matching Property** is enforced via the `prevLogIndex`/`prevLogTerm` fields sent with every `AppendEntries` RPC:

1. The leader sends the new entry along with the index/term of the entry that should immediately precede it in the follower's log.
2. The follower checks whether it actually has an entry at that index with that term.
3. If not, it rejects the RPC.
4. On rejection, the leader decrements the index it's trying to sync from and retries — walking backward through the log until it finds a point where the follower's log matches the leader's.
5. From that point forward, the leader overwrites any conflicting entries in the follower's log with its own and replicates forward again until the follower is caught up.

This mechanism guarantees that if two logs contain an entry with the same index and term, the logs are identical in every entry up to that point — which is what lets the system safely reconcile a rejoining or lagging node without manual intervention.

## 7. Raft's Safety Guarantees

Beyond the mechanics above, the Raft paper formally guarantees five safety properties that hold at all times, even across leader crashes and network partitions:

- **Election Safety** — at most one leader can be elected in a given term.
- **Leader Append-Only** — a leader never overwrites or deletes entries in its own log; it only appends.
- **Log Matching** — if two logs contain an entry with the same index and term, all preceding entries are identical in both logs.
- **Leader Completeness** — if an entry is committed in a given term, it will be present in the logs of all future leaders (enforced by the election restriction in §4).
- **State Machine Safety** — if a node has applied a particular log entry to its state machine, no other node will ever apply a different command for that same log index.

Together, these properties are what let you reason about a Raft-backed system as if it were a single, reliable machine — even though it's really running on multiple, individually unreliable ones.

## 8. Generalizing Beyond Databases

Raft is most commonly described in terms of replicating a database, but the underlying abstraction is more general: a **replicated state machine**. Any deterministic finite state machine — not just a database — can be kept consistent across a cluster this way. As long as every node starts from the same initial state and applies the same sequence of committed log entries, every node arrives at an identical resulting state, deterministically.

This is why Raft (or Paxos-family variants) shows up underneath so many different kinds of distributed systems, not just databases:
- Distributed SQL databases: **CockroachDB**, **YugabyteDB**, **TiDB/TiKV**
- Coordination services: **etcd** (which itself backs Kubernetes), **HashiCorp Consul**
- Distributed logs and messaging systems that need ordered, durable commit logs

## Glossary

| Term | Meaning |
|---|---|
| Log | Ordered, append-only sequence of commands stored on each node |
| Term | A logical "epoch" of time, started by an election; used to detect stale leaders |
| Leader | The single node handling writes and driving replication for the current term |
| Follower | A passive node that accepts entries and heartbeats from the leader |
| Candidate | A node temporarily campaigning for leadership during an election |
| Committed entry | A log entry acknowledged by a majority of nodes; guaranteed durable |
| Election timeout | Randomized interval after which a follower starts an election if it hasn't heard from a leader |
| `AppendEntries` RPC | The message type used both for heartbeats and for replicating log entries |
| `RequestVote` RPC | The message type a candidate sends to solicit votes during an election |

## Further Reading

- Ongaro & Ousterhout, *"In Search of an Understandable Consensus Algorithm"* (the original Raft paper) — the definitive source for the full protocol, including cluster membership changes and log compaction, which aren't covered above.
- The Raft visualization at **raft.github.io** is a good companion for building intuition about elections and log replication interactively.
