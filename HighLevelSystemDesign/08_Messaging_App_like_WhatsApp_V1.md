# Chat System (WhatsApp / MS Teams Style) — System Design Interview Guide
### Senior & Staff Engineer Level

---

## Table of Contents

1. [Problem Statement & Scope](#1-problem-statement--scope)
2. [Reference Architecture Gap Analysis](#2-reference-architecture-gap-analysis)
3. [Requirements](#3-requirements)
4. [Capacity Estimation](#4-capacity-estimation)
5. [High-Level Architecture](#5-high-level-architecture)
6. [Component 1: Connection Gateway (WebSocket Fleet)](#6-component-1-connection-gateway-websocket-fleet)
7. [Component 2: Session Registry & Presence Service](#7-component-2-session-registry--presence-service)
8. [Component 3: Message Service — Sequencing & Idempotency](#8-component-3-message-service--sequencing--idempotency)
9. [Component 4: Durable Log Backbone (Kafka)](#9-component-4-durable-log-backbone-kafka)
10. [Component 5: Group Chat Fan-out](#10-component-5-group-chat-fan-out)
11. [Component 6: Offline Queue & Multi-Device Delivery](#11-component-6-offline-queue--multi-device-delivery)
12. [Component 7: Push Notification Service](#12-component-7-push-notification-service)
13. [Component 8: Media / Asset Pipeline](#13-component-8-media--asset-pipeline)
14. [Component 9: User & Group Metadata Service](#14-component-9-user--group-metadata-service)
15. [Component 10: End-to-End Encryption — The Anchor Constraint](#15-component-10-end-to-end-encryption--the-anchor-constraint)
16. [Component 11: Ephemeral Fast Path (Typing & Read Receipts)](#16-component-11-ephemeral-fast-path-typing--read-receipts)
17. [CAP / PACELC Positioning](#17-cap--pacelc-positioning)
18. [Failure Modes & Mitigations](#18-failure-modes--mitigations)
19. [Scalability & Sharding](#19-scalability--sharding)
20. [Senior vs Staff Answer Differentiators](#20-senior-vs-staff-answer-differentiators)
21. [Interview Time Allocation](#21-interview-time-allocation)
22. [Quick-Reference Cheatsheet](#22-quick-reference-cheatsheet)
23. [Related Guides](#23-related-guides)

---

## 1. Problem Statement & Scope

A **chat system** is a distributed system that lets users exchange messages (text, media, structured events) in real time, across 1:1 and group conversations, while remaining available and mostly correct across unreliable mobile networks, multiple simultaneous devices, and server failures.

"WhatsApp-style" and "Teams-style" are not the same problem wearing different skins — they diverge on a genuine **anchor constraint**:

| | WhatsApp-style (consumer) | Teams-style (enterprise) |
|---|---|---|
| Identity | Phone number | Corporate directory (AAD/LDAP) |
| Default privacy posture | End-to-end encrypted by default | Encrypted in transit/at rest; **not** E2EE by default |
| Server's view of content | Ciphertext only — cannot read, index, or moderate | Plaintext accessible — search, DLP, compliance, eDiscovery, legal hold |
| Retention | Ephemeral — delete after delivery is the historical default | Indefinite — regulatory retention requirements |
| Structure | Flat 1:1 / group chats | Channels, threads, tabs, persistent team history |

Naming this constraint explicitly, before drawing a single box, is a Staff-level move: it determines whether the server can do content search, spam/abuse detection, rich link previews, or message editing/recall — none of which are free once you've committed to E2EE. This guide builds a WhatsApp-leaning design by default and calls out at each component where a Teams-style deployment would diverge.

### What the interviewer is really testing

- Can you route a message from sender to recipient in real time when both are behind independent, horizontally-scaled connection servers?
- Do you understand **durability-before-delivery** — never losing a message even if it can't be routed live?
- Can you reason about **multi-device fan-out**, not just single-connection-per-user?
- Do you separate **durable** (messages) from **ephemeral** (typing, presence) data paths?
- Can you name the CAP/PACELC tradeoff **per component**, not for "the system" as a whole?
- Do you proactively raise group fan-out, encryption, and offline delivery — or wait to be asked?

---

## 2. Reference Architecture Gap Analysis

Two commonly-referenced course diagrams anchor this guide: a detailed Educative-style diagram (explicit WebSocket manager, Mnesia, Kafka, Redis) and a simplified ByteByteGo-style diagram (Chat servers + Presence servers as a "Real time service," API servers, Notification servers, generic KV stores). Both are legitimate starting points; neither is Staff-complete on its own. A useful interview habit is to explicitly gap-analyze whatever reference architecture you're handed before extending it — treat this the way you'd treat any existing document you're asked to improve.

### Diagram A (Educative-style — detailed)

**What it gets right:**
- Splits the stateful WebSocket layer from stateless backend services behind a second internal load balancer — correct separation of concerns.
- A dedicated **WebSocket manager + Redis cluster** for connection routing is exactly the right primitive (this guide keeps it, renamed "Session Registry").
- **Mnesia** as the per-user message store is a direct, accurate nod to WhatsApp's real Erlang/BEAM heritage — Mnesia is Erlang's built-in distributed database, and BEAM's actor model is precisely why a single WhatsApp box could historically hold ~1–2 million concurrent connections. If you recognize and can explain *why* that detail is there, it's a strong signal; most candidates copy the box without knowing what it means.
- **Kafka** in front of the Group Message Handler is the right instinct for durable fan-out.
- Media (CDN + Blob storage) is correctly kept out of the message hot path.
- User service and Group service each own their data (Redis cache + MySQL), which is proper service-oriented data ownership.

**Gaps (what a Staff answer must add):**
- **No presence service.** Online/offline/last-seen is folded implicitly into the WebSocket manager, but presence fan-out (broadcasting a status change to every contact) is a distinct, expensive problem — it deserves its own box and its own scaling story (Section 7).
- **No encryption layer at all.** For a WhatsApp-style system this is not a minor omission — it's the single biggest architectural driver (Section 15).
- **No explicit message ordering / sequencing mechanism.** Two different message IDs are needed (global uniqueness vs. per-conversation order) and the diagram is silent on both.
- **No idempotency / deduplication.** At-least-once delivery without a dedup key produces duplicate messages on retry.
- **Unlabeled components.** The small "database with a lightning bolt" icons attached to each WebSocket server are never explained. In an interview, an unlabeled box on your own diagram invites the exact question you don't want to improvise an answer to — always be able to say what it is (here, the most defensible read is a local outbound buffer for in-flight, unacknowledged pushes; state that explicitly rather than leaving it ambiguous).
- **Single-connection-per-user framing.** There is no visible support for a user logged in on phone + desktop + web simultaneously, which is table stakes for both WhatsApp and Teams today (Section 11).
- **No fan-out strategy split.** Every group appears to go through one path regardless of size; a 3-person group and a 50,000-subscriber broadcast channel should not be architected identically (Section 10).

### Diagram B (ByteByteGo-style — simplified)

**What it gets right:**
- **Presence servers are a first-class, separate box** from Chat servers — better than Diagram A on this specific point, and this guide adopts that separation.
- Clean split between the stateless **http** path (API servers) and stateful **ws** path (Real time service) at the load-balancer level.
- **Notification servers** are decoupled from the message path — correct, since push delivery is a different failure domain (third-party APNs/FCM outages shouldn't touch core messaging).
- Each service owns its own KV store — no shared-database coupling.

**Gaps (what a Staff answer must add):**
- **The hardest problem in the whole system is invisible.** When User A (connected to Chat Server 1) messages User B (connected to Chat Server 2), how does Server 1 find Server 2? The diagram has three "Chat servers" boxes with no routing mechanism between them — this is precisely the WebSocket-manager/Session-Registry problem Diagram A gets right and Diagram B omits.
- **No durable log.** Messages appear to go directly into "KV store" with no Kafka-equivalent, which makes replay, at-least-once redelivery, and audit/compliance retention (mandatory for the Teams-style variant) hard to reason about.
- **No group chat / fan-out service** shown anywhere.
- **No media pipeline.** No CDN, no blob storage, no distinct upload path.
- **"KV store" ×3 is under-specified.** Three identical icons with no partitioning key, replication factor, or consistency model named is a Senior-level gap — a Staff answer names the technology and the CAP choice for each one individually (Section 17).
- **No ordering, dedup, or encryption**, same as Diagram A.

### The synthesis this guide builds

Take Diagram B's clean **service separation** (presence separate from chat, notifications separate from API) and graft on Diagram A's **connection-routing primitive** (Session Registry) and **durable log** (Kafka), then add the four pieces both diagrams omit: encryption, ordering/idempotency, multi-device fan-out, and a size-aware group fan-out strategy. That synthesis is Sections 5–16.

---

## 3. Requirements

### Functional Requirements

| Requirement | Notes |
|---|---|
| 1:1 messaging | Core primitive |
| Group messaging | Up to ~1,000 members (WhatsApp-style); larger "channel"/broadcast case handled separately |
| Delivery status | Sent → Delivered → Read, per recipient device |
| Typing indicators | Ephemeral, best-effort |
| Presence | Online / offline / last-seen |
| Multi-device | Same account active on phone + desktop + web concurrently |
| Media sharing | Images, video, voice notes, files |
| Push notifications | When no active connection / app backgrounded |
| History sync | New device login retrieves recent history (bounded by retention policy) |
| Message search | Out of scope for deep-dive; briefly note server-side search is incompatible with default E2EE |

### Non-Functional Requirements

| Requirement | Target (illustrative) |
|---|---|
| Scale | ~2B monthly users, ~100B messages/day processed |
| Latency | p99 < 200ms sender→recipient when recipient is online |
| Availability | 99.99%+ |
| Durability | Zero message loss once server has acknowledged the sender |
| Ordering | Per-conversation causal order preserved on the client |
| Delivery semantics | At-least-once + idempotent apply (not exactly-once) |
| Security | E2EE by default (WhatsApp anchor) *or* encrypted-at-rest + auditable (Teams anchor) — pick one explicitly |

---

## 4. Capacity Estimation

All figures below are **illustrative order-of-magnitude numbers**, not verified production figures — treat them as a vehicle for showing the math, not as claims about any real company's infrastructure.

```
Assume: 100 billion messages/day processed (sends + deliveries combined)

Throughput:
  100,000,000,000 / 86,400 ≈ 1.16M messages/sec (avg)
  Peak (3x, diurnal + regional skew) ≈ 3.5M messages/sec

Average wire size per message (text + envelope + E2EE overhead):
  ≈ 1 KB  (payload ~100-200B; ratchet header, MAC, IDs, timestamps add the rest)

Ingest bandwidth:
  1.16M/s × 1KB ≈ 1.16 GB/s (sender → server, before fan-out)

Concurrent WebSocket connections:
  ~500M DAU, ~20-25% concurrently online at peak ≈ 100-125M concurrent connections

Gateway node sizing — the runtime is itself a capacity lever:
  Conventional stack (Netty/Go), ~50K conns/node:
      125M / 50,000 ≈ 2,500 gateway nodes
  Erlang/BEAM actor-per-connection (WhatsApp's historical real stack), ~1-2M conns/node:
      125M / 1,000,000 ≈ 125 gateway nodes
  → A 20x difference in fleet size from runtime choice alone. Naming this
    tradeoff (throughput-per-node vs. operational familiarity/ecosystem)
    is a stronger signal than picking either stack silently.

Session Registry sizing:
  ~125M active (user, device) → server mappings, ~100B/entry ≈ 12.5 GB
  → comfortably fits a modest Redis cluster with headroom for growth

Durable message store (raw, pre-replication):
  100B msgs/day × 1KB ≈ 100 TB/day  →  ×3 replication ≈ 300 TB/day
```

**Key insight (do not miss this one):** the anchor constraint from Section 1 changes the capacity math itself, not just the security story. A WhatsApp-style system that deletes messages from its servers once every recipient device has acknowledged delivery keeps only a rolling **offline queue**, not a permanent archive — its steady-state storage footprint is bounded by "undelivered backlog," not by total message volume. A Teams-style system with compliance retention keeps **everything**, indefinitely, and its dominant cost curve looks like the Web Crawler guide's storage argument: storage, not compute, is the long-run bottleneck. State which regime you're in before sizing storage — it's a 100x-or-more difference.

**Key insight #2:** group fan-out multiplies delivery volume far more than sender-side ingest does. If even 15-20% of messages are sent to groups averaging 20-30 members, group deliveries can rival or exceed all 1:1 delivery traffic combined. This is why Group Fan-out gets its own component with its own scaling strategy (Section 10) rather than being a code path inside the Message Service.

---

## 5. High-Level Architecture

### Real-time message path

```
                         ┌────────────────────────┐
                         │   Client (multi-device: │
                         │   phone · desktop · web)│
                         └─────┬──────────────┬────┘
                        WS/TLS │              │ HTTPS (profile, contacts, media)
                    ┌──────────▼───────┐  ┌────▼─────────────┐
                    │  L4 Load Balancer│  │  API Gateway      │
                    │  (WS, connection-│  │  (stateless HTTP) │
                    │   aware routing) │  └────┬──────────────┘
                    └──────────┬───────┘       │
                    ┌──────────▼───────────┐   │
                    │ Connection Gateway    │   │
                    │ Fleet (stateful WS)   │   │
                    │ — Component 1 —       │   │
                    └───┬───────────────┬───┘   │
              register/ │               │ deliver
              heartbeat │               │
                    ┌───▼───────────────▼───┐  ┌▼─────────────────┐
                    │ Session Registry &     │  │ User & Group      │
                    │ Presence Service       │  │ Metadata Service  │
                    │ (Redis, TTL) — Comp. 2 │  │ (sharded SQL) — 9 │
                    └───────────┬────────────┘  └───────────────────┘
                                │ lookup: user → {device: server}
                    ┌───────────▼────────────┐
                    │   Message Service       │
                    │   seq alloc · idempotent│
                    │   apply · E2EE envelope │
                    │   pass-through — Comp. 3│
                    └───────────┬─────────────┘
                                │ durable append (ack sender only AFTER this)
                    ┌───────────▼─────────────┐
                    │   Kafka (durable log)    │
                    │   partition: conversation│
                    │   _id — Component 4      │
                    └──┬────────────────────┬──┘
                       │                    │
             ┌─────────▼──────┐   ┌─────────▼─────────┐
             │ Group Fan-out   │   │ Offline Queue Store│
             │ Service         │   │ per (user, device) │
             │ — Component 5   │   │ — Component 6      │
             └─────────┬───────┘   └─────────┬───────────┘
                       │                     │
                       │           ┌─────────▼─────────┐
                       │           │ Push Notification  │
                       │           │ Service → APNs/FCM │
                       │           │ — Component 7       │
                       │           └────────────────────┘
                       │
           delivers back via Session Registry lookup → Connection Gateway → client
```

### Media path (kept out of the message hot path)

```
Client → Asset Service → Blob Store (S3/GCS) ⇄ CDN
             (message itself carries only a reference/key, never the bytes)
```

### Ephemeral path (no durability, no offline queue — see Component 11)

```
Client → Connection Gateway ⇄ Redis Pub/Sub (typing, read receipts) ⇄ Connection Gateway → Client
```

**Kafka is the seam**, exactly as in the Web Crawler and Ordered Message Logging guides: it decouples ingestion from fan-out/delivery, gives replay for reconnecting offline devices, and provides the audit trail a Teams-style deployment needs for compliance.

---

## 6. Component 1: Connection Gateway (WebSocket Fleet)

### What it does

A stateless-business-logic, stateful-connection fleet. Each node holds many open sockets but should own no durable data of its own — everything durable lives in the Session Registry, the durable log, or the offline queue.

### Responsibilities

```
On connect:
  1. Authenticate (JWT / session token)
  2. Register (user_id, device_id) → this server's id in Session Registry, with TTL
  3. Start heartbeat (client ping every ~20-30s; renews TTL)

On inbound client message:
  4. Forward to Message Service (internal RPC) — gateway does not persist it itself

On outbound delivery:
  5. Message Service / Fan-out Service looks up recipient's server via Session Registry
  6. Pushes to the specific gateway node hosting that (user, device) socket
  7. Gateway writes to the socket; buffers briefly if the write is in flight

On disconnect / TTL expiry:
  8. Deregister from Session Registry (or let TTL lapse — treat absence as offline)
```

### Reconnection is the common case, not the exception

Mobile clients flap between wifi/cellular constantly. Design reconnect as a **cheap, cursor-based resume**, not a full re-sync:

```
Client reconnect payload: { device_id, last_acked_seq_per_conversation }
Server: replay only messages after last_acked_seq, from the durable log or offline queue
```

A full re-sync on every reconnect is the single most common Senior-level performance bug in this design.

### Why route through a gateway fleet instead of connecting clients directly to backend services?

Because the socket itself is stateful (it lives on exactly one machine at a time) while everything behind it should not be. This isolates the one genuinely stateful layer and lets every other service scale independently and be restarted without dropping connections it doesn't own.

---

## 7. Component 2: Session Registry & Presence Service

### Session Registry

```
Key:   user_id
Value: [ { device_id, server_id, connected_at, last_heartbeat } , ... ]
Store: Redis cluster, partitioned by hash(user_id)
TTL:   renewed on each heartbeat; expiry = implicit offline
```

This is the mechanism that answers Diagram B's unaddressed question: "how does Chat Server 1 find Chat Server 2?" — it doesn't; it asks the registry, which knows exactly which gateway node (if any) currently holds a socket for the recipient's each device.

### Presence is a distinct, harder problem than "is the socket open"

A naive design broadcasts every connect/disconnect to all of a user's contacts — O(contacts) fan-out on every state change. At scale this is the same shape as the celebrity/hot-key fan-out problem in the News Feed guide: a user with thousands of contacts triggers thousands of pushes on every phone unlock.

**Staff-level mitigation — hybrid push/pull:**
- **Push** presence updates only to contacts with whom the user has a currently **open** conversation view (a small, bounded set).
- **Pull** presence on-demand when a contact list or chat is opened, rather than maintaining a live subscription for every contact all the time.
- Debounce/hysteresis on the "online" signal — a 2-second network blip shouldn't flicker a contact's status; require N missed heartbeats before flipping to offline, and consider a brief "last seen recently" fuzzy state instead of hard binary online/offline.

### CAP note

Presence is a canonical **AP** component (see Section 17) — a few seconds of staleness is a non-issue, and blocking a connect on cross-region presence consistency would be a self-inflicted latency wound.

---

## 8. Component 3: Message Service — Sequencing & Idempotency

### Two different IDs solve two different problems — do not conflate them

This mirrors a trap named in this project's cross-cutting principles for consensus-quorum vs. replication-quorum: two concepts that sound similar and are easy to merge in an answer, but serve entirely different purposes.

| ID | Purpose | Properties needed |
|---|---|---|
| `message_id` | Global uniqueness, dedup key, roughly time-sortable | Snowflake-style (node + timestamp + counter) — see Distributed Unique ID Generation guide |
| `conversation_seq` | Client-side rendering order within one conversation | Strictly monotonic **per conversation**, gap-detectable |

A single ID cannot cleanly do both: a globally-unique Snowflake ID is not strictly monotonic per-conversation once multiple gateway nodes can accept sends for the same conversation concurrently, and a strictly-monotonic per-conversation counter is not globally unique or useful as a dedup key across conversations.

### Idempotency (at-least-once, not exactly-once)

```
Client generates client_msg_id (UUID) at send time, before any network call.
On timeout, client retries the send with the SAME client_msg_id.
Server dedups on (sender_id, client_msg_id) before creating a new message_id.
```

This is the same "don't chase exactly-once, chase at-least-once + idempotent apply" framing used throughout the Kafka and Web Crawler guides. Exactly-once delivery across an unreliable mobile network is not a tractable target; idempotent retries are.

### Ack ordering — the durability-before-delivery rule

**Do not acknowledge the sender until the message is durably appended to the log (Section 9), not merely until it's been pushed to a recipient's socket.** Acking on socket write is a classic bug: the gateway can crash after the socket write but before durable persistence, silently losing a message the sender believes was delivered. Ack-after-durable-append is the correct order even though it adds a hop of latency.

---

## 9. Component 4: Durable Log Backbone (Kafka)

Every accepted message is appended to Kafka **before** the sender is acknowledged. This single decision underpins nearly everything downstream:

- **Replay** for a device that reconnects after being offline for hours.
- **Fan-out source of truth** for the Group Fan-out Service.
- **Audit trail** for a Teams-style compliance/eDiscovery requirement (this is exactly why the durable log is non-optional for that variant, even though a pure WhatsApp deployment could in principle discard it faster).

### Partitioning choice

```
Topic: message-events
Partition key: conversation_id
  + preserves in-order delivery within a conversation (a single partition
    is consumed in order)
  - risk: a single very active group is a single hot partition

Mitigation for hot groups: sub-partition by (conversation_id, shard_of_N)
  once a conversation's message rate crosses a threshold — the same
  hot-key sub-queue pattern used for hot domains in the Web Crawler guide.
```

Durability contract: `acks=all` + a `min.insync.replicas` set to at least 2 is the actual guarantee, not `acks` alone — see the Kafka guide for why conflating the two is a known interview slip.

---

## 10. Component 5: Group Chat Fan-out

### Fan-out-on-write vs. fan-out-on-read — size-aware, not one-size-fits-all

This is the same tradeoff explored in the News Feed / Twitter Fan-out guide, applied to groups instead of followers:

| Group size | Strategy | Why |
|---|---|---|
| Small–medium (up to ~1,000, typical WhatsApp/Teams group) | **Fan-out-on-write**: expand membership at send time, publish one delivery event per member | Bounded fan-out cost; recipients get low-latency push |
| Very large (broadcast channels, distribution lists — tens of thousands to millions of subscribers) | **Fan-out-on-read**: publish once to a shared topic; subscribers pull/subscribe to the topic rather than receiving N individual writes | Avoids a single send becoming a million-write storm — the group-chat analogue of the celebrity fan-out problem |

Neither reference diagram makes this distinction — treating every group identically is a Senior-level gap this guide corrects.

### Membership consistency at send time

A message sent while a membership change (add/remove) is in flight can otherwise produce "who was actually in the group when this was sent" ambiguity. Attach a **versioned membership snapshot** (or at minimum the sender's view of the member list at send time) to each fan-out event, and give new joiners a `joined_at` cursor so they see history from their join point forward, not retroactively — matching real WhatsApp/Teams behavior and avoiding a subtle correctness bug.

---

## 11. Component 6: Offline Queue & Multi-Device Delivery

### Per-(user, device) queue, not per-user

```
Store: Cassandra / DynamoDB-style wide-column store (see Distributed KV
       Store guide) — partition key (user_id, device_id)
Entry: { message_id, conversation_id, payload_ref, enqueued_at }
Cleared: on delivery ack from that specific device
TTL: bounded (e.g. 30 days) — beyond that, notify the client of a gap
     rather than growing the queue unboundedly
```

### Multi-device is an anchor-constraint choice, not a footnote

Two real architectures exist here, and naming which one you're building is a Staff-level move:

1. **Mirror architecture** (WhatsApp's historical model): the phone is the primary; companion devices (web/desktop) relay through it. Simple, but if the phone is offline, companion devices lose functionality too.
2. **Fully server-mediated multi-device** (modern WhatsApp's actual public direction, and Teams' native model): every device is a first-class, independently-registered participant with its own offline queue and its own encryption identity. No single device is a bottleneck, at the cost of needing to fan out to N devices per recipient instead of 1, and — under E2EE — needing to distribute keys to N devices per sender too (Section 15).

This guide recommends the fully server-mediated model as the default improvement over what either reference diagram implies (both read as single-connection-per-user).

---

## 12. Component 7: Push Notification Service

Kept fully decoupled from the message plane — this is the one place Diagram B is already correct and worth preserving as-is.

```
Trigger: recipient has zero active gateway connections for a given device
         (Session Registry lookup comes back empty)
Talks to: APNs (iOS) / FCM (Android) / WNS (desktop, Teams)
Not the source of truth: a dropped push is not a lost message — the
  offline queue (Component 6) is what guarantees eventual delivery once
  the client reconnects on its own.
```

### E2EE constrains the push payload

The push payload cannot contain plaintext message content under an E2EE anchor — it can only be a wake-up signal ("you have N new messages"). The client fetches and decrypts the actual content over its own authenticated channel after waking. Wrap calls to third-party push providers behind a circuit breaker (see Circuit Breaker Pattern guide) — an APNs/FCM outage must degrade to "delayed delivery on reconnect," not cascade into gateway or message-service failures. See also the standalone Notification System guide for provider-specific retry/backoff detail this guide doesn't re-derive.

---

## 13. Component 8: Media / Asset Pipeline

Matches the strongest part of both reference diagrams — media never enters the WebSocket/message hot path:

```
Client → Asset Service → Blob Store (S3/GCS)
Asset Service returns a reference (URL or storage key)
Client sends a normal message whose payload is that reference
Downloads served via CDN, not through the message service
```

Under E2EE, the client encrypts media **before** upload; the server stores an opaque encrypted blob and never holds a plaintext decryption key — the key travels inside the E2EE message envelope itself. Malware/content scanning of uploads (relevant primarily for the Teams-style, non-E2EE variant, since an E2EE server cannot inspect plaintext content) is a separate concern — see the File Upload & Malware Scanning guide rather than re-deriving it here.

---

## 14. Component 9: User & Group Metadata Service

Standard CRUD service: sharded relational store (MySQL/Postgres) by `user_id`/`group_id` hash, Redis cache in front for read-heavy profile and contact lookups — this is the one area both reference diagrams already get essentially right. See the Sharding Strategies guide for shard-key and resharding tradeoffs rather than re-deriving them here.

---

## 15. Component 10: End-to-End Encryption — The Anchor Constraint

Neither reference diagram addresses encryption at all — for a WhatsApp-style system, this is the single largest gap to close, because it reshapes what the server is even allowed to do.

### The high-level mechanism (Signal Protocol family)

```
Key agreement:  X3DH — lets two parties establish a shared secret
                asynchronously, without both being online at once,
                using pre-key bundles the server stores and hands out.
Ongoing session: Double Ratchet — derives a new encryption key per
                message, giving forward secrecy (a compromised key
                doesn't expose past messages) and post-compromise
                security (future messages recover after a compromise).
```

### What this does to the server's role

The server becomes **a relay and a durable store for ciphertext it cannot read.** Consequences that ripple through earlier sections:

- No server-side content search or spam/content moderation on message bodies (only on metadata: rate, fan-out patterns, reports).
- No server-generated rich link previews from message text.
- Push payloads carry no plaintext (Section 12).
- Group encryption doesn't scale pairwise Double Ratchets to hundreds of members — real systems use **sender-key** schemes (one symmetric key per sender per group, distributed once, rotated on membership change) rather than deriving the full protocol here.
- **Multi-device compounds directly with encryption**: every linked device is a distinct cryptographic identity that must be added to every ongoing session's key list. This is precisely why WhatsApp's real multi-device rollout was reported as a multi-year engineering effort — it's not "add another socket," it's "re-key every active conversation for a new participant identity."

### The Teams-style divergence, stated plainly

An enterprise deployment that needs eDiscovery, DLP, and legal hold **cannot** be fully E2EE by default and still meet those requirements, because they require the platform (not just the two endpoints) to be able to read content. This is the anchor constraint from Section 1 made concrete: state which regime you're designing for, then let it drive search, retention, and moderation decisions consistently rather than contradicting yourself component-by-component.

---

## 16. Component 11: Ephemeral Fast Path (Typing & Read Receipts)

### Why these need a second, separate pipe

Typing indicators and read receipts are high-frequency and low-value-if-lost — routing them through the durable log + offline queue (Components 4 and 6) would add write amplification for signals nobody needs redelivered after the fact. A dropped typing indicator is a UI non-event, not a support ticket.

```
Client → Connection Gateway ⇄ Redis Pub/Sub (or a short-retention Kafka
         topic) ⇄ Connection Gateway → Client

No Kafka durable log. No offline queue. No delivery guarantee.
If recipient is offline, the event is simply dropped — by design.
```

### The delivery-status state machine still needs *some* durability

Read receipts feel ephemeral but the underlying **delivery state** (sent → delivered → read) is not purely ephemeral — the sender's UI needs it to persist across their own reconnects. Split the concern: the *live* typing/presence-style ping is pure ephemeral pub/sub; the *delivery-state transition* itself is a small, idempotent event that does go through the normal Message Service path (Component 3) so it survives a sender reconnect. Stating this split explicitly — rather than treating "read receipts" as one undifferentiated ephemeral blob — is itself a Staff-level signal: the "two-speed" durable/ephemeral design is a deliberate architectural choice, not an afterthought.

---

## 17. CAP / PACELC Positioning

Per-component, not system-wide — most of this system is AP/EL, with a small number of deliberate CP exceptions.

| Component | CAP / PACELC | Reasoning |
|---|---|---|
| Session Registry (Redis) | **AP / EL** | A few seconds of stale routing info is recoverable via reconnect; never worth blocking on. |
| Presence | **AP / EL** | Staleness is invisible to users at the timescales that matter. |
| Durable log (Kafka) | **AP under partition, EL normally**, but tuned for durability via `acks=all` + `min.insync.replicas` | Availability for writes to reachable replicas; never silently drop an acked message. |
| Conversation sequence allocation | **CP** | A single-writer-per-partition (or a fenced epoch/term, per Leader Election guide) is required — split-brain here produces genuinely ambiguous message order, not just staleness. |
| Offline queue (Cassandra/Dynamo-style) | **AP / EL**, tunable quorum | See Distributed KV Store guide for the `W+R>N` replication-quorum tradeoff — distinct from the consensus quorum used for sequence allocation above; conflating the two is a known slip. |
| User/Group metadata — writes | **CP-leaning** for the primary (membership changes shouldn't split-brain) | |
| User/Group metadata — reads | **AP** via read replicas | Scale reads without coupling them to primary consistency. |
| E2EE key server (identity keys, pre-key bundles) | **CP** | Stale or inconsistent key material doesn't just degrade UX — it breaks the cryptographic handshake outright. This is a genuinely load-bearing CP exception, not a generic "coordination needs consistency" example. |
| Push notification dispatch | **AP** | Best-effort by definition; the offline queue is the real guarantee. |
| Ephemeral pub/sub (typing, presence pings) | **AP, no durability at all** | Dropping is the correct behavior under partition, not a failure. |

---

## 18. Failure Modes & Mitigations

| Failure | Symptom | Mitigation |
|---|---|---|
| Gateway node crash | Sockets on that node drop | Client auto-reconnect with cursor-based resume (Section 6); Session Registry TTL expiry naturally clears stale routing |
| Ack-on-socket-write bug | Sender believes message delivered; it wasn't durably stored | Enforce ack-after-durable-append ordering (Section 8) as a hard invariant |
| Duplicate delivery on retry | User sees the same message twice | Idempotency key dedup at Message Service and defensively again at client render layer |
| Session Registry entry stale (ghost "online") | Sender pushes to a socket that's gone | Heartbeat TTL + gateway push failure triggers immediate re-lookup and offline-queue fallback |
| Hot group / large broadcast channel | One send becomes a write storm | Switch to fan-out-on-read above a size threshold (Section 10) |
| Kafka partition lag | Fan-out falls behind ingestion | Backpressure: slow ingest acceptance or scale the fan-out consumer group; monitor consumer lag directly |
| Sequence allocator split-brain | Two nodes both believe they own the next seq for a conversation | Single-writer-per-partition, or fenced epoch tokens (Leader Election guide) |
| Push provider outage (APNs/FCM down) | Notifications delayed | Circuit breaker around the provider call; offline queue guarantees eventual delivery on client reconnect regardless |
| Clock skew across nodes | `message_id` ordering assumptions break if ID relies on wall clock alone | Snowflake-style IDs use a monotonic per-node counter + coordinated epoch, not raw wall-clock time (Distributed Unique ID Generation guide) |
| Multi-device key desync (E2EE) | A newly-linked device can't decrypt / is missing from a session | Explicit device-add protocol re-keys affected sessions rather than assuming automatic propagation |
| Group membership race during send | Ambiguity over who was "in" the group at send time | Versioned membership snapshot attached to the fan-out event; `joined_at` cursor for new members |
| Offline queue unbounded growth | A permanently-offline device accumulates forever | TTL-bound the queue; notify client of a gap requiring fuller re-sync beyond the TTL window |

---

## 19. Scalability & Sharding

| Component | Scaling strategy |
|---|---|
| Connection Gateway | Horizontally add nodes; no shared state beyond registry entries; runtime choice (Erlang/BEAM vs. conventional) directly sets conns-per-node (Section 4) |
| Session Registry | Partition by `hash(user_id)` across a Redis cluster |
| Kafka | Partition by `conversation_id`; sub-partition hot conversations (Section 9) |
| Group Fan-out | Size-aware split between write-fan-out and read-fan-out paths (Section 10) |
| Offline Queue | Partition by `(user_id, device_id)` |
| User/Group Metadata | Shard by `user_id`/`group_id` hash — see Sharding Strategies guide for resharding mechanics |
| Media | S3/GCS + CDN scale independently and are effectively unbounded from this system's perspective |
| Multi-region | Regional gateway deployment with geo-routing; durable log replicated cross-region so a device can reconnect to its nearest region rather than always its "home" region — flagged as a real complexity source, not derived in depth here |

---

## 20. Senior vs Staff Answer Differentiators

### Senior-level expectations

- Correctly identify the core components: gateway, message service, storage, notifications.
- Explain how a WebSocket connection maps a user to a server.
- Mention Kafka or an equivalent durable queue.
- Handle the obvious failure mode (gateway crash → reconnect).
- Produce basic capacity math.

### Staff-level expectations (additional)

| Topic | What Staff-level adds |
|---|---|
| Anchor constraint | Names the E2EE-vs-compliance tradeoff up front and lets it drive search/retention/moderation decisions consistently |
| Message IDs | Separates global-uniqueness ID from per-conversation sequence explicitly; doesn't conflate them |
| Delivery semantics | States at-least-once + idempotent apply as the deliberate target, not a gap to be embarrassed about |
| Presence | Hybrid push/pull fan-out to avoid O(contacts) broadcast storms |
| Group fan-out | Size-aware fan-out-on-write vs. fan-out-on-read split |
| Multi-device | Names it as an anchor-constraint choice (mirror vs. fully server-mediated) with real cost implications, including on E2EE key distribution |
| Encryption | Explains Signal-protocol-family mechanics at the right depth and what it forecloses server-side |
| Ephemeral vs. durable | Explicit two-speed design: typing/presence never touch the durable log; delivery-state transitions do |
| CAP/PACELC | Per-component, including the load-bearing CP exception for E2EE key material — not a single system-wide label |
| Failure modes | Proactively raised (ack-ordering bug, split-brain sequencing, hot-group storms) before being asked |
| Reference-diagram critique | Can spot what's missing or ambiguous in *any* diagram handed to them — including unlabeled components — rather than accepting it uncritically |

### The single most important Staff differentiator

**Treat every reference architecture — including ones handed to you by the interviewer — as a draft to gap-analyze, not a spec to reproduce.** Section 2 of this guide is that habit applied explicitly: both source diagrams are reasonable starting points, and the improved design is what's left after naming precisely what each one is missing and why.

---

## 21. Interview Time Allocation

For a 45-minute system design session:

| Phase | Time | Focus |
|---|---|---|
| Requirements & anchor constraint | 5 min | Confirm 1:1 + group scope; state the E2EE-vs-compliance choice up front |
| Capacity estimation | 5 min | Messages/sec, concurrent connections, gateway sizing, storage regime |
| High-level architecture | 5 min | Draw gateway, session registry, message service, Kafka, fan-out, offline queue |
| Connection routing & Session Registry (deep dive) | 8 min | How Server A finds Server B; presence fan-out mitigation |
| Message ordering, idempotency, durability-before-delivery | 8 min | Two-ID split; ack ordering; at-least-once framing |
| Group fan-out | 6 min | Size-aware write vs. read fan-out |
| Encryption & multi-device | 5 min | Signal-protocol-family at a high level; what it forecloses |
| Failure modes | 3 min | Ack-ordering bug, split-brain sequencing, hot-group storm |

**What to cut if short on time:** media pipeline depth (both diagrams already get this mostly right) and multi-region detail. **Never cut:** connection routing, ordering/idempotency, or the encryption anchor-constraint discussion — these are where both reference diagrams are weakest and where Staff signal is highest.

---

## 22. Quick-Reference Cheatsheet

```
KEY MECHANISMS
───────────────
Connection routing:  Session Registry (Redis)  |  user → {device: server}
Message uniqueness:  Snowflake-style message_id  |  dedup key
Message ordering:    per-conversation monotonic conversation_seq (SEPARATE from message_id)
Idempotency:         client_msg_id generated client-side, dedup on (sender_id, client_msg_id)
Durability rule:     ack sender only AFTER durable log append, never on socket write alone
Group fan-out:       write fan-out (small/med groups)  |  read fan-out (large broadcast channels)
Encryption:          X3DH (async key agreement) + Double Ratchet (per-message keys, forward secrecy)
Ephemeral path:      typing/presence via Redis Pub/Sub — no Kafka, no offline queue, drop is OK

KEY NUMBERS (illustrative)
────────────────────────────
100B messages/day        → 1.16M msg/sec avg, ~3.5M msg/sec peak
~125M concurrent conns   → 125 gateway nodes (Erlang/BEAM) or ~2,500 (conventional stack)
1KB avg wire size        → ~1.16 GB/s ingest bandwidth
100TB/day raw messages   → ×3 replication ≈ 300TB/day (only if retaining everything — Teams-style)

KEY SYSTEMS
────────────
Connection layer:    Gateway fleet (stateful WS) + L4 LB
Routing:             Redis (Session Registry)
Durable log:          Kafka, partitioned by conversation_id
Offline queue:        Cassandra/Dynamo-style, partitioned by (user_id, device_id)
Metadata:             Sharded MySQL/Postgres + Redis cache
Media:                S3/GCS + CDN, kept out of the message hot path
Push:                 APNs / FCM / WNS, behind a circuit breaker

CAP DECISIONS
──────────────
AP/EL:  Session Registry, Presence, Offline Queue, Metadata reads, Push dispatch, Ephemeral pub/sub
CP:     Conversation sequence allocation, E2EE key server, Metadata primary writes

STAFF-LEVEL SIGNALS TO HIT
─────────────────────────────
✓ Name the E2EE-vs-compliance anchor constraint before drawing boxes
✓ Two distinct IDs: global message_id vs. per-conversation seq
✓ Ack-after-durable-append, never ack-on-socket-write
✓ Size-aware group fan-out (write vs. read)
✓ Multi-device as a first-class design choice, tied explicitly to E2EE key distribution
✓ Ephemeral (typing/presence) vs. durable (messages) as two deliberately separate pipes
✓ Per-component CAP/PACELC, including the CP exception for key material
✓ Proactively critique any reference diagram handed to you — name what's missing before extending it
```

---

## 23. Related Guides

- **Kafka** — durability contract (`acks` + `min.insync.replicas`), partitioning mechanics referenced in Components 4 and 9.
- **Ordered Message Logging** — the general ordering/log-replay pattern this guide's durable log leans on directly.
- **Distributed Unique ID Generation** — Snowflake-style `message_id` construction (Component 3).
- **Distributed Key-Value Store (Dynamo-style)** — replication-quorum tradeoffs for the offline queue (Component 6); kept explicitly distinct from consensus quorum in Section 17.
- **Leader Election in Distributed Systems** — fencing/epoch tokens for the conversation sequence allocator (Component 3, Section 18).
- **Sharding Strategies** — shard-key and resharding mechanics for metadata and the durable log.
- **CAP/PACELC Theorem** — general framework this guide applies per-component in Section 17.
- **Circuit Breaker Pattern** — wrapping calls to APNs/FCM and other third-party push providers (Component 7).
- **News Feed System / Twitter Fan-out Architecture** — the fan-out-on-write vs. fan-out-on-read tradeoff this guide applies to group size in Component 5.
- **Notification System** — provider-specific push retry/backoff detail not re-derived here.
- **File Upload & Malware Scanning** — content scanning for uploaded media in the non-E2EE (Teams-style) variant.
- **Web Crawler** — origin of the "storage, not compute, is the long-run cost driver" and hot-partition sub-sharding framings reused in Sections 4 and 9.

---

*Guide compiled for Senior & Staff-level system design interview preparation.*
*Topics: Chat Systems, WhatsApp, Microsoft Teams, WebSockets, End-to-End Encryption, Message Ordering, Fan-out.*
