# Messaging App (WhatsApp / MS Teams) — System Design Interview Guide
### Senior & Staff Engineer Level

---

## Table of Contents

1. [Problem Statement & Scope](#1-problem-statement--scope)
2. [Requirements](#2-requirements)
3. [Capacity Estimation](#3-capacity-estimation)
4. [High-Level Architecture](#4-high-level-architecture)
5. [The Anchor Constraint Fork: E2EE-First vs Compliance-First](#5-the-anchor-constraint-fork-e2ee-first-vs-compliance-first)
6. [Component 1: Connection Gateway](#6-component-1-connection-gateway)
7. [Component 2: Message Ingestion, IDs & the Ordering Sequencer](#7-component-2-message-ingestion-ids--the-ordering-sequencer)
8. [Component 3: Fanout Service](#8-component-3-fanout-service)
9. [Component 4: Message Store](#9-component-4-message-store)
10. [Component 5: Delivery State Machine & Read Receipts](#10-component-5-delivery-state-machine--read-receipts)
11. [Component 6: Multi-Device Sync](#11-component-6-multi-device-sync)
12. [Component 7: End-to-End Encryption](#12-component-7-end-to-end-encryption)
13. [Component 8: Presence Service](#13-component-8-presence-service)
14. [Component 9: Media Pipeline](#14-component-9-media-pipeline)
15. [Component 10: Push Notification Bridge](#15-component-10-push-notification-bridge)
16. [Component 11: Search](#16-component-11-search)
17. [Component 12: Group/Channel Membership & Permissions](#17-component-12-groupchannel-membership--permissions)
18. [Component 13: Call Signaling (WebRTC)](#18-component-13-call-signaling-webrtc)
19. [CAP / PACELC Positioning](#19-cap--pacelc-positioning)
20. [Failure Modes & Mitigations](#20-failure-modes--mitigations)
21. [Scalability & Sharding](#21-scalability--sharding)
22. [Senior vs Staff Answer Differentiators](#22-senior-vs-staff-answer-differentiators)
23. [Interview Time Allocation](#23-interview-time-allocation)
24. [Quick-Reference Cheatsheet](#24-quick-reference-cheatsheet)

---

## 1. Problem Statement & Scope

A **messaging platform** delivers real-time text, media, and call communication between individuals and groups, across multiple devices per user, at global scale, with strong delivery and ordering guarantees despite unreliable mobile networks and intermittent connectivity.

This guide anchors on two real products because they force genuinely different architectures from the same starting point:

- **WhatsApp** — consumer, end-to-end encrypted (E2EE) by default, 1:1 and group chats, server is deliberately blind to content.
- **MS Teams** — enterprise, compliance-first (eDiscovery, legal hold, DLP), channels/threads, deep calendar/file/call integration, content generally **not** E2EE (encrypted in transit and at rest, but server-readable for compliance and search).

### What the interviewer is really testing

- Can you design low-latency, high-fanout pub/sub at billions-of-messages/day scale?
- Do you treat **delivery guarantees and ordering** as first-class design problems, not details?
- Can you apply **CAP/PACELC per component**, including the subtle case of the ordering sequencer?
- Do you recognize that "messaging app" forks into materially different systems depending on one upstream decision (E2EE vs server-visible content) — and can you trace that fork through search, moderation, backup, and admin tooling?

---

## 2. Requirements

### Functional Requirements

| Requirement | Notes |
|---|---|
| Send/receive 1:1 and group messages | Text, images, video, voice notes, files |
| Delivery states per recipient (and per device) | sent → delivered → read |
| Multi-device support | Same account live on phone + desktop + web simultaneously |
| Presence | Online / last-seen / typing indicators |
| Push notifications | For backgrounded/offline devices |
| History sync on new device | Full or windowed backfill |
| Group & channel management | Create/add/remove members, roles; Teams adds Team → Channel → Thread hierarchy |
| Voice/video calls | 1:1 and group (Teams especially) |
| In-conversation search | Text and media metadata |
| (Teams) Compliance | eDiscovery, legal hold, DLP, retention policies |
| (WhatsApp) End-to-end encryption | Server never sees plaintext |

### Non-Functional Requirements

| Requirement | Target |
|---|---|
| Scale | ~2B users, ~100B messages/day (illustrative, WhatsApp-scale order of magnitude) |
| Latency | p99 send-to-deliver < 100ms when recipient is online |
| Availability | 99.99%+; **message loss is unacceptable**, duplicate delivery is tolerable (client dedups) |
| Ordering | Strict **per-conversation** ordering; no cross-conversation ordering guarantee |
| Durability | Message durably persisted before ack to sender |
| Global | Multi-region; enterprise tenants may require data residency |

---

## 3. Capacity Estimation

```
Assumptions:
  2B monthly active users
  ~100B messages/day  (order-of-magnitude, not a fact to defend)
  ~500M peak concurrent persistent connections

Throughput:
  100,000,000,000 / 86,400 ≈ 1.16M messages/sec (average)
  Peak (3x for regional/event spikes) ≈ 3.5M messages/sec

Message envelope size (ciphertext + headers: sender, conversation_id,
seq_no, MAC, timestamps) ≈ 1KB
  Text-plane bandwidth: 1.16M/s × 1KB ≈ 1.16 GB/s

Media (≈15% of messages carry media, avg 300KB):
  174K/s × 300KB ≈ 52 GB/s
  → media MUST bypass the message-ingestion hot path; direct-to-object-store
    upload with the message carrying only a reference (see §14)

Concurrent connections:
  500M peak ÷ ~75K persistent connections per gateway node (epoll-based WS)
  ≈ 6,667 nodes → provision ~8,000 with headroom

Storage:
  Message metadata + ciphertext: 100B/day × 1KB ≈ 100 TB/day
  Media: 174K/s × 86,400s × 300KB ≈ ~4.5 PB/day (dominant cost by far)

Push notifications:
  ~25% of messages target an offline/backgrounded device
  25B/day ≈ 289K/sec average, highly bursty by region/business-hours
```

**Key insight:** Just like the crawler's storage was dominated by raw HTML, here the **message-plane** (text, metadata, ordering) and the **media-plane** (images/video/files) have wildly different cost and latency profiles and must be architected as **two separate pipelines** — the text plane optimizes for latency and ordering; the media plane optimizes for throughput and cost, and is fetched out-of-band via CDN-backed object storage.

---

## 4. High-Level Architecture

```
┌──────────┐    ┌──────────┐    ┌──────────┐
│ Client A │    │ Client B │    │ Client C │   (phone/desktop/web — multi-device)
└────┬─────┘    └────┬─────┘    └────┬─────┘
     │ WebSocket/MQTT persistent conn  │
     └────────────────┼─────────────────┘
                       ▼
           ┌───────────────────────┐
           │  Connection Gateway   │  (session registry: user_id/device_id → gateway node)
           │  (thousands of nodes) │
           └───────────┬───────────┘
                       │  message submitted
                       ▼
           ┌───────────────────────┐
           │  Message Ingestion    │  assign message_id (dedup key)
           │  Service              │
           └───────────┬───────────┘
                       │  Kafka, partition key = conversation_id
                       ▼
           ┌───────────────────────┐
           │  Ordering Sequencer   │  per-conversation seq_no
           │  (= Kafka partition   │  (single leader per partition = free CP)
           │   leader + ISR)       │
           └───────────┬───────────┘
                       │
        ┌──────────────┼───────────────┬──────────────────┐
        ▼               ▼               ▼                  ▼
 ┌─────────────┐ ┌─────────────┐ ┌──────────────┐  ┌───────────────┐
 │ Message     │ │ Fanout      │ │ Push Bridge  │  │ Search Indexer │
 │ Store       │ │ Service     │ │ (APNs/FCM)   │  │ (Teams only)   │
 │ (Cassandra) │ │             │ │              │  │                │
 └─────────────┘ └──────┬──────┘ └──────────────┘  └───────────────┘
                        │  route by session registry
                        ▼
              back to Connection Gateway → recipient devices
```

**Kafka** (partitioned by `conversation_id`) sits between ingestion and everything downstream — same decoupling, replay, and back-pressure rationale as in the crawler design, but here the partition also **doubles as the ordering primitive** (see §7).

---

## 5. The Anchor Constraint Fork: E2EE-First vs Compliance-First

This is the single most consequential decision in the whole system, and it forks the architecture at nearly every layer:

| Layer | WhatsApp (E2EE-first) | MS Teams (Compliance-first) |
|---|---|---|
| Content visibility | Server stores ciphertext only; cannot read plaintext | Server stores/reads plaintext (encrypted at rest, not E2EE) |
| Search | Client-side local index only; server searches metadata only | Server-side full-text index (Elasticsearch), required for eDiscovery |
| Spam/abuse detection | Metadata-only heuristics (rate, graph, reports) — no content scanning | Full content-based DLP and moderation possible |
| Backup | Encrypted client-side; server can't recover lost keys | Server-side backup/retention with legal-hold guarantees |
| Multi-device | Each device is an independent E2EE session endpoint (pairwise or sender-keys) | Devices are just additional read/write clients against server-held plaintext |
| Admin tooling | Group admin, but no message-content admin capability | Full admin visibility: legal hold, retention policy, conditional access |

**Staff-level framing:** don't present this as "WhatsApp uses encryption and Teams doesn't." Both encrypt in transit and at rest. The fork is **whether the server itself is inside or outside the trust boundary**. Every component from here forward (§12 encryption, §16 search, §17 permissions) should trace back to this one decision — exactly like the Samsung Pay guide's design tracing to zero-network tap-to-pay and treat-device-as-compromised.

---

## 6. Component 1: Connection Gateway

### Transport choice

| Option | Pros | Cons |
|---|---|---|
| **WebSocket** | Full-duplex, low overhead after handshake, broad browser/mobile support | Requires sticky connection management; proxies/firewalls occasionally interfere |
| **MQTT** | Purpose-built pub/sub for constrained mobile networks; QoS levels built in | Extra protocol to operate; less native browser support (needs WS bridge) |
| **Long polling** | Works through any HTTP infra, trivially compatible | Higher latency, higher connection churn and server load at this scale |

**Recommendation:** WebSocket (or MQTT over WebSocket for mobile-optimized QoS) as primary transport, with long-polling as a compatibility fallback. This mirrors the crawler's "cheap path vs specialized fleet" pattern (§8 of the crawler guide) — most clients use the efficient path; a fallback exists for constrained environments.

### Session Registry

```
Registry entry (Redis Cluster, or Dynamo-style KV — see Distributed KV Store guide):
  key:   user_id:device_id
  value: { gateway_node_id, connected_at, last_heartbeat }
  TTL:   refreshed on heartbeat (~30s); expires if node goes silent

Write path:   AP — a stale registry entry just causes one extra fanout
              retry hop, not data loss. Use R/W quorum tuned for fast
              writes (W=1, R=1) — same reasoning as the Dynamo-style KV
              guide's quorum trade-off, deliberately NOT linearizable.
```

### Staff-Level Gotcha

**Reconnect storms.** A regional outage or app-wide crash causes millions of clients to reconnect within seconds. Naive reconnect floods the gateway tier and the session registry simultaneously.

**Mitigation:** exponential backoff with jitter on the client, plus **admission control** at the gateway (rate-limit new-connection acceptance per node, shed load gracefully) rather than accepting connections until the node falls over. This is the direct analog of the crawler's DNS-storm mitigation — bound the blast radius of a mass simultaneous event.

---

## 7. Component 2: Message Ingestion, IDs & the Ordering Sequencer

### Two different IDs, two different jobs

A common Senior-level slip is conflating these:

| ID | Purpose | Generated by | Properties |
|---|---|---|---|
| `message_id` | **Idempotency / dedup key** | Client (UUID) or server (Snowflake-style — see Unique ID Generation guide) | Globally unique, no ordering meaning |
| `conversation_seq` | **Ordering within one conversation** | Kafka partition (offset) or a dedicated per-shard counter | Monotonically increasing, meaningful *only* within one conversation |

Retrying a failed send re-uses the same `message_id`; the server dedups on it. Ordering is decided independently and later, by whichever replica currently leads that conversation's partition.

### Why a dedicated sequencer is usually unnecessary

**Staff-level insight — this is the most important architectural call in the whole system:** if you partition Kafka by `conversation_id`, the **partition leader + ISR already gives you a CP-scoped sequencer for free** — exactly the same durability contract the Kafka guide covers (`acks=all` + `min.insync.replicas ≥ 2`). You don't need to hand-roll a Raft group per conversation; you need to configure Kafka's existing consensus primitive correctly and route on the right key.

```
Producer:
  partition_key = hash(conversation_id) % num_partitions
  → all messages for one conversation land on one partition
  → one leader per partition = single writer = natural total order
  → Kafka offset within that partition IS the conversation_seq

Durability contract (same lesson as the Kafka guide):
  acks=all + min.insync.replicas=2  → an acked message survives a
  single broker failure without silent data loss or reordering.
```

**When you DO need a bespoke sequencer:** if ordering must be assigned **synchronously before ack** with sub-partition-swap consistency guarantees during a leader election window, or if you need cross-partition transactional ordering (rare for chat). For the interview, default to "Kafka partition = sequencer," and only escalate to a dedicated Raft-backed sequencer service if the interviewer pushes on leader-failover ordering gaps.

### Ingestion pipeline

```
1. Client sends message over gateway connection (message_id, conversation_id, ciphertext)
2. Ingestion service validates, checks message_id dedup cache (short-TTL, Redis)
3. Produce to Kafka, key = conversation_id
4. On produce ack (min.insync.replicas satisfied) → ack sender: "sent"
5. Downstream consumers (fanout, store, push, search) consume independently
```

---

## 8. Component 3: Fanout Service

Same fundamental tension as the Twitter Fan-out guide: **fanout-on-write vs fanout-on-read.**

| Approach | Pros | Cons | Best for |
|---|---|---|---|
| **Fanout-on-write** (push to every member's queue immediately) | Low read latency; simple client model | O(members) write amplification; brutal for huge groups | 1:1 chats, small/medium groups (the overwhelming majority) |
| **Fanout-on-read** (recipient pulls from shared conversation log) | O(1) write regardless of group size | Higher read latency; client must pull/poll or hold a long-lived cursor | Huge Teams channels (10K+ members), WhatsApp Communities/broadcast lists |
| **Hybrid** | Push for online devices, log-based pull for offline catch-up and for oversized groups | More moving parts | Production systems at this scale — this is what's actually built |

```
Fanout decision per conversation:
  if member_count ≤ threshold (e.g. 1,000):
      fanout-on-write → look up each member's active devices in the
      session registry → push directly to their gateway connections
  else:
      fanout-on-read → write once to the conversation log; members
      pull on connect / poll a cursor; presence-online members get a
      lightweight "new message" ping, not the full payload
```

This is the same "celebrity problem" the Twitter guide solves for follower fanout, applied to channel membership instead of follower graphs — worth saying explicitly if you've covered that guide, since it signals the pattern transfers.

---

## 9. Component 4: Message Store

| Layer | System | Contents | Why |
|---|---|---|---|
| Per-conversation timeline | Cassandra (or similar wide-row store) | Ciphertext, seq_no, sender, timestamp | Partition by `conversation_id`; matches Kafka partitioning; efficient "give me messages N..M for this conversation" |
| Delivery/read cursors | Redis or the same wide-row store | Per-device last-delivered/last-read seq_no | High write throughput; ephemeral-ish, doesn't need long-term durability guarantees beyond current state |
| Media references | Object store pointers only (see §14) | URL/blob-id + encryption key wrapper, never the media itself | Keeps the hot message-plane row small |

```sql
CREATE TABLE conversation_messages (
  conversation_id   TEXT,
  seq_no            BIGINT,
  message_id        TEXT,
  sender_id         TEXT,
  ciphertext        BLOB,
  sent_at           TIMESTAMP,
  PRIMARY KEY (conversation_id, seq_no)
) WITH CLUSTERING ORDER BY (seq_no ASC);
```

Partitioning by `conversation_id` collocates a conversation's full history — same rationale as the crawler's `url_metadata` table partitioned by domain.

### Retention

```
WhatsApp:   messages typically NOT retained server-side after delivery
            to all devices (server is a relay, not an archive) —
            direct consequence of the E2EE anchor constraint.
Teams:      retained per org policy, often indefinitely under legal
            hold — direct consequence of the compliance anchor constraint.
```

---

## 10. Component 5: Delivery State Machine & Read Receipts

```
State machine (per message, per recipient DEVICE):

  SENT ──(ingestion ack)──▶ DELIVERED ──(client ack on receipt)──▶ READ

  SENT:      server has durably persisted the message (Kafka acked)
  DELIVERED: a specific recipient device has received it over the wire
  READ:      the user has viewed it on that device

Conversation-level "delivered"/"read" (what the sender sees) is an
aggregate: delivered when ALL of the recipient's active devices have
delivered it; read when the read-marking device reports READ (and
apps typically simplify further with "read = read on ANY device").
```

### Staff-Level Gotcha: read-receipt fanout amplification

In a group of N members, naively fanning out "user X read message M" to all N-1 other members is **O(N²)** traffic for the conversation. At Teams-channel scale (thousands of members) this collapses the fanout tier.

**Mitigation:** aggregate read state ("Read by 47") and batch/debounce receipt fanout — don't push a receipt event per reader in real time for large groups; push periodic aggregate updates instead. Same shape of fix as the crawler's Bloom-filter fill-ratio monitoring: detect the amplifying case and switch strategy rather than applying one algorithm uniformly.

---

## 11. Component 6: Multi-Device Sync

A user may have phone + desktop + web simultaneously logged in. Two design questions:

1. **How does a new device catch up on history?** Windowed backfill from the message store (Teams: server has plaintext, trivial) or from an **encrypted backup** the device decrypts locally (WhatsApp: server never holds plaintext, so backfill is either peer-device-to-device transfer or an E2EE cloud backup the user controls).
2. **How is a live message delivered to all of a user's devices at once?** Fanout treats `user_id` as a **set of device endpoints**, not a single target — each device has its own session-registry entry and, under E2EE, its own encryption session (see §12).

```
Fanout target resolution:
  user_id → [device_1 (online, gateway A), device_2 (offline), device_3 (online, gateway B)]

  → deliver immediately to device_1 and device_3
  → queue for device_2, trigger push notification (§15)
  → device_2 backfills its missed-message gap on next connect via
    (seq_no watermark it last acked) .. (current seq_no)
```

---

## 12. Component 7: End-to-End Encryption

**This section only fully applies on the WhatsApp side of the fork (§5).** Teams typically uses TLS in transit + at-rest encryption with server-held keys, not true E2EE, precisely because compliance requires server-side plaintext access.

### Signal Protocol / Double Ratchet (WhatsApp-style)

```
Per-device-pair session state:
  Identity key    — long-term, per device
  Signed prekey   — medium-term, rotated periodically
  One-time prekeys — single-use, consumed on first session establishment

Double Ratchet:
  Every message advances a symmetric-key ratchet AND, periodically, a
  Diffie-Hellman ratchet → forward secrecy (a compromised key doesn't
  expose past messages) and post-compromise security (future messages
  self-heal after a compromised key is rotated out).

Groups:
  Pairwise Double Ratchet doesn't scale to N members (O(N²) sessions).
  Production systems use "sender keys": each sender maintains one
  symmetric ratchet shared (via pairwise-encrypted setup) with all
  group members — O(N) not O(N²).
```

### Staff-Level Gotcha: prekey exhaustion / replay

One-time prekeys are **single-use by design** — reusing one breaks forward secrecy for that session and opens a replay attack surface.

```
Prekey server operation MUST be atomic fetch-and-delete:
  BAD:  read prekey → return it → delete it   (race: two concurrent
        session-establishment requests can both read the same prekey
        before either delete completes)
  GOOD: atomic conditional delete-and-return (e.g. Redis Lua script,
        or a CP-scoped compare-and-swap) — this is a CP operation
        even though the surrounding system is otherwise AP.

Prekey pool depletion: monitor per-device remaining one-time-prekey
count; auto-replenish from the device when online; fall back to the
signed prekey alone (weaker forward secrecy) with an alert if a
device is offline long enough to exhaust its pool.
```

This is exactly the kind of **adversarial, domain-specific trap** (like spider traps or decompression bombs in the crawler/scanning guides) that signals Staff-level thinking — the failure mode is a security property, not a throughput number.

---

## 13. Component 8: Presence Service

```
State:      online | last_seen_ts | typing (per conversation, ephemeral)
Storage:    Redis pub/sub + TTL keys — NOT the durable message store
Write path: heartbeat on the gateway connection refreshes TTL; typing
            events are fire-and-forget with a short TTL (a few seconds)
Fanout:     only to conversation participants who are actively viewing
            that conversation (avoid broadcasting typing globally)
```

**Senior-level mistake:** persisting every presence change to a durable database. At this write rate (every heartbeat, every typing keystroke debounce) that's a self-inflicted write-amplification failure. Presence is inherently ephemeral and belongs in a **pure AP, TTL-based, non-durable** store.

---

## 14. Component 9: Media Pipeline

```
Upload:
  1. Client requests a pre-signed upload URL from the media service
  2. Client uploads directly to object storage (S3/GCS) — bypasses the
     message-ingestion hot path entirely (this is why §3's 52 GB/s
     media bandwidth doesn't hit the text-plane infrastructure)
  3. (WhatsApp) media is encrypted client-side before upload; the
     message itself carries only the blob reference + decryption key,
     both inside the E2EE ciphertext
  4. (Teams) media can be scanned server-side for malware/DLP before
     being made available — see the File Upload & Malware Scanning
     guide for the scanning pipeline this reuses directly

Download:
  CDN-fronted object storage; thumbnails generated async and cached
  separately from full-resolution originals (tiered like the
  crawler's raw-HTML hot/cold storage tiering)
```

---

## 15. Component 10: Push Notification Bridge

Reuses the general Notification System guide's architecture directly: this service is a consumer of the fanout pipeline that targets **offline/backgrounded** devices via APNs (iOS) / FCM (Android) instead of a live gateway connection.

### Staff-Level Gotcha: push storms on mass reconnect

If a regional outage takes many users offline simultaneously, queued messages trigger a push-notification burst on recovery that can itself get rate-limited or throttled by APNs/FCM.

**Mitigation:** coalesce multiple pending messages per device into a single "you have N new messages" push rather than one push per message; respect per-app platform rate limits; prioritize push dispatch by conversation recency, not FIFO across the whole backlog.

---

## 16. Component 11: Search

Direct consequence of §5's anchor-constraint fork:

| | WhatsApp (E2EE) | Teams (compliance) |
|---|---|---|
| Index location | **Client-side only** — local on-device index (e.g. SQLite FTS) built from decrypted plaintext the client already holds | **Server-side** — Elasticsearch/similar, built from plaintext the server already holds |
| What the server can search | Metadata only (sender, date, media type, conversation) | Full message content, across the org, for eDiscovery |
| Cross-device search | Each device rebuilds/syncs its own local index | Single server-side index serves all clients and admin tooling |

**Staff-level framing:** don't propose a server-side content search index for a system you've just said is E2EE — that's an internal contradiction the interviewer is listening for. If asked "how does WhatsApp search work," the correct answer is "it doesn't, server-side — it can't, by design."

---

## 17. Component 12: Group/Channel Membership & Permissions

Teams-specific hierarchy: **Team → Channel → Thread**, with roles (owner/member/guest) and channel visibility (standard/private/shared).

```
Membership store: strongly consistent for security-relevant mutations
  (add/remove member, change role) — this must NOT be eventually
  consistent, because a stale cache means a removed member can still
  read/post after removal.

Read-mostly metadata (channel name, topic, pinned messages): fine to
  cache aggressively and serve AP-style; low security sensitivity.
```

### Staff-Level Gotcha: stale-ACL access after removal

**A cached membership/permission entry that outlives an actual removal is a security trap**, not just a staleness inconvenience — directly analogous to how the crawler guide treats spider traps and the scanning guide treats decompression bombs as domain-specific adversarial concerns that must be named explicitly, not left implicit.

**Mitigation:** permission checks on message *send* and *fetch* consult a short-TTL, actively-invalidated cache (push invalidation on membership change, not just TTL expiry) — this is one of the few places in an otherwise AP-leaning system where you deliberately pay a consistency cost.

---

## 18. Component 13: Call Signaling (WebRTC)

Voice/video is architecturally almost a **separate subsystem** from store-and-forward messaging — same "separate fleet" principle as the crawler's headless-browser fleet for JS-rendered pages.

```
Signaling: SDP offer/answer + ICE candidates relayed over the existing
           gateway connection (reuse the transport, not the messaging
           semantics — no ordering/durability guarantees needed here,
           it's ephemeral session setup)

NAT traversal: STUN (discover public address) → TURN (relay fallback
           when direct P2P isn't reachable)

Media path (NOT via the messaging backend at all):
  1:1 calls:    peer-to-peer where possible
  Group calls:  SFU (Selective Forwarding Unit) — forwards streams
                without decoding/re-encoding; scales far better than
                a full mesh (O(N) links per participant, not O(N²))
                or an MCU (which trades bandwidth for CPU-heavy mixing)
```

**Senior-level answer:** "add WebRTC for calls."
**Staff-level answer:** calls have their own signaling plane, their own media infrastructure (SFU fleet), and explicitly do **not** flow through the message store/sequencer/fanout pipeline built for text — conflating them is a modeling error.

---

## 19. CAP / PACELC Positioning

| Component | Under Partition (CAP) | Else / Normal Operation (PACELC) | Reasoning |
|---|---|---|---|
| Connection/session registry | **AP** | EL (favor latency) | Stale gateway pointer costs one retry hop, not data loss |
| Ordering sequencer (Kafka partition) | **CP**, scoped per conversation | EC (favor consistency once partition heals) | This is the CP island — exactly like the Proximity Service's driver-assignment CP island — everything else in the system stays AP |
| Message store (Cassandra) | **AP** | EL, tunable | QUORUM read when a device needs its own recent history reliably; ONE for high-throughput writes |
| Presence | **AP** | EL | Staleness of a few seconds is invisible to users; never worth blocking on |
| Prekey server (one-time prekey consumption) | **CP** | EC | Atomic fetch-and-delete required — the one deliberately consistent slice of an otherwise AP encryption layer |
| Membership/permissions (mutations) | **CP** | EC | Stale ACL = security hole, not just UX staleness (§17) |
| Membership/permissions (read-mostly metadata) | **AP** | EL | Channel name/topic staleness is harmless |
| Media object store | **AP** | EL | Append-only blobs; availability strongly preferred |
| Push bridge | **AP** | EL | Best-effort by nature (APNs/FCM themselves are best-effort) |
| Search index (Teams) | **AP** | EL, with monitored lag | Slightly stale search results are acceptable; index lag is observable and bounded |

**The single most important CAP decision in this system:** almost everything is AP — except two narrow, deliberately-chosen CP islands: the **per-conversation ordering sequencer** and **security-critical mutations** (prekey consumption, ACL changes). Naming these two islands explicitly, and explaining *why* only these two need consistency while everything else can be AP, is the core Staff-level signal for this guide — structurally identical to how the Dynamo-style KV Store guide keeps R/W quorum and consensus quorum distinct, applied here at the component-selection level instead.

---

## 20. Failure Modes & Mitigations

| Failure | Symptom | Detection | Mitigation |
|---|---|---|---|
| Gateway node crash | Connected clients drop | Connection count / heartbeat miss | Externalize session state (registry, not in-memory only); client reconnects with a resume cursor (last-acked seq_no), not a full resync |
| Partial fanout (some devices delivered, not all) | Recipient sees message on phone, not desktop | Per-device delivery cursor lag | At-least-once retry via the fanout queue; client-side dedup on `message_id` |
| Duplicate delivery | Same message shown twice | Client observes repeat `message_id` | Client-side dedup — never rely on exactly-once delivery |
| Out-of-order delivery (multi-path fanout) | Group messages arrive out of sequence | Gap in `conversation_seq` on client | Client buffers and reorders by `seq_no`; gap triggers a backfill fetch, not a display-as-received |
| Sequencer partition leader failure | Brief unavailability for that conversation shard | Kafka ISR/leader-election metrics | `min.insync.replicas ≥ 2`; minority partition rejects writes; short CP pause preferred over silent reordering |
| Reconnect storm after regional outage | Gateway/registry overload | Connection-accept rate spike | Exponential backoff + jitter on client; admission control on gateway |
| Push notification storm | APNs/FCM throttling | Push dispatch error rate | Coalesce multiple pending messages into one push per device; rate-limit per platform policy |
| Large-group fanout amplification | Fanout tier saturates for one huge channel | Fanout queue depth per conversation | Hybrid fanout-on-read for oversized groups (§8) |
| Read-receipt amplification | O(N²) receipt traffic in large groups | Receipt fanout volume per conversation | Aggregate/batch receipts beyond a member-count threshold (§10) |
| Prekey exhaustion/replay | Session establishment fails or reuses a key | Prekey pool depletion metric | Atomic fetch-and-delete; auto-replenish; alert on depletion (§12) |
| Stale ACL after member removal | Removed member still reads/posts | Permission-check audit / anomaly detection | Push-invalidated short-TTL permission cache, not pure TTL expiry (§17) |
| Clock skew across devices/regions | Naive timestamp-based ordering breaks | Recrawl-style drift detection on send timestamps | Never order by wall-clock; use `conversation_seq` exclusively (same lesson as the crawler's clock-skew mitigation) |
| Media upload abuse (oversized/malicious files) | Storage blowup or malware distribution | Upload size/type anomalies | Size limits + malware scanning pipeline (reuses File Upload & Malware Scanning guide) |
| Kafka/fanout consumer lag | Delivery latency climbs | Consumer lag monitoring | Scale fanout consumer fleet; apply backpressure; prioritize online-user delivery over offline backlog |

---

## 21. Scalability & Sharding

```
Message plane sharding:
  shard_key = hash(conversation_id) % num_partitions
  → Kafka partition, Cassandra partition, and (if used) sequencer
    shard all use the SAME key → one conversation's full pipeline
    stays co-located, preserving ordering without cross-shard
    coordination

Connection plane sharding:
  Gateway nodes are stateless w.r.t. routing — session registry
  (Redis Cluster / Dynamo-style KV, consistent hashing) maps
  user_id:device_id → current gateway node. Registry itself shards
  by consistent hashing across the KV cluster, same pattern as the
  Distributed KV Store guide.

Horizontal scaling:
| Component            | Strategy                                          |
|-----------------------|---------------------------------------------------|
| Connection gateway     | Add nodes; stateless; registry tracks assignment  |
| Kafka (message plane)  | Add partitions/brokers; repartition by conv_id    |
| Message store          | Add nodes; consistent hash ring; RF=3             |
| Fanout consumers       | Add nodes; scale with consumer-group parallelism  |
| Push bridge            | Add nodes; stateless; rate-limited per platform   |
| Media object store     | Managed; scales automatically                     |
| Search index (Teams)   | Add data nodes; increase shard count              |
```

### Cross-region

```
Pin each conversation to a "home region" (based on majority of
members) for ingestion + sequencing → minimizes the common case
latency. Members outside the home region read via async
cross-region replication — bounded staleness for them, full
consistency for the majority. Enterprise tenants (Teams) may
override this with hard data-residency pinning regardless of
member distribution — a compliance requirement trumping the
latency optimization.
```

---

## 22. Senior vs Staff Answer Differentiators

### Senior-level expectations

- Identify core components: gateway, message store, fanout, push notifications
- Know that WebSockets are used for real-time delivery
- Mention encryption in transit and at rest
- Describe basic 1:1 and group messaging
- Handle obvious failure modes (server crash, retry on failure)
- Provide basic capacity math

### Staff-level expectations (additional)

| Topic | What Staff-level adds |
|---|---|
| Architecture fork | Proactively name the E2EE-vs-compliance anchor constraint and trace it through search, admin, backup, moderation |
| IDs | Distinguish `message_id` (dedup) from `conversation_seq` (ordering) — never conflate them |
| Ordering | Recognize Kafka partition leadership as a free, correctly-scoped CP sequencer instead of building a bespoke one |
| Fanout | Hybrid fanout-on-write/fanout-on-read by group size, not one strategy for all conversations |
| Multi-device | Model `user_id` as a *set* of device endpoints, each with independent session state under E2EE |
| Encryption | Double Ratchet + sender-keys for groups; atomic prekey consumption as a named security-critical CP operation |
| Presence | Ephemeral TTL/pub-sub store, explicitly never the durable message store |
| Read receipts | Recognize and mitigate O(N²) fanout amplification in large groups |
| Permissions | Treat stale-ACL-after-removal as a security trap requiring push invalidation, not passive TTL |
| Calls | Separate signaling + SFU media plane, not routed through the messaging backend |
| CAP/PACELC | Name the two deliberate CP islands (sequencer, security-critical mutations) against an otherwise AP system, with reasoning for each |
| Failure modes | Proactively enumerate reconnect storms, push storms, prekey exhaustion, ACL staleness — before being asked |

### The single most important Staff differentiator

**Tracing every downstream decision back to the E2EE-vs-compliance anchor constraint, and correctly identifying the two narrow CP islands (ordering sequencer, security-critical mutations) inside an otherwise AP system.** A Senior answer treats messaging as "pub/sub plus encryption." A Staff answer shows that almost the entire hard part of this system is knowing exactly where consistency is non-negotiable and being deliberately loose everywhere else.

---

## 23. Interview Time Allocation

For a 45-minute system design session:

| Phase | Time | Focus |
|---|---|---|
| Requirements & scope | 4 min | Confirm WhatsApp-style vs Teams-style; functional + non-functional |
| Capacity estimation | 4 min | Messages/sec, connections, bandwidth split (text vs media) |
| High-level architecture | 5 min | Draw all components; name Kafka as the backbone |
| Anchor constraint fork (if relevant) | 2 min | Call out E2EE vs compliance if the interviewer hasn't specified |
| Connection gateway & sequencer (deep dive) | 10 min | Transport choice, session registry, ID vs seq_no, Kafka-as-sequencer insight |
| Fanout & delivery | 8 min | Fanout-on-write vs read, delivery state machine, multi-device |
| Encryption (if WhatsApp-style) | 5 min | Double Ratchet, sender keys, prekey atomicity |
| Failure modes | 5 min | Reconnect storms, push storms, ACL staleness, read-receipt amplification |
| Calls / search (if time) | 2 min | Flag as separate subsystems; don't over-invest |

**What to cut if short on time:** call signaling detail and search internals. **Never cut:** the ordering sequencer discussion and the delivery/ordering failure modes — that's where most of the Staff signal lives.

---

## 24. Quick-Reference Cheatsheet

```
KEY DECISIONS
─────────────
Transport:        WebSocket (MQTT-over-WS for mobile QoS), long-poll fallback
Ordering:         Kafka partition per conversation_id = free CP sequencer
IDs:              message_id (dedup, global) ≠ conversation_seq (order, per-conv)
Fanout:           fanout-on-write (small/medium) | fanout-on-read (huge channels)
Encryption:        Double Ratchet (1:1) + sender keys (groups) — WhatsApp-style only
Presence:          Redis pub/sub + TTL, never the durable store
Calls:             Separate signaling + SFU media plane, not the messaging backend

KEY NUMBERS
───────────
100B msgs/day        → 1.16M msgs/sec avg, ~3.5M/sec peak
500M peak connections → ~8,000 gateway nodes (75K conns/node)
Media: ~52 GB/s        → bypasses text-plane hot path entirely
100 TB/day text+meta   → Cassandra, partitioned by conversation_id
~4.5 PB/day media      → object store, dominant cost

CAP DECISIONS
─────────────
AP (default): session registry, message store, presence, media store,
              push bridge, search index, read-mostly permissions
CP (deliberate islands only):
   • Ordering sequencer — scoped PER CONVERSATION, not system-wide
   • Prekey consumption (atomic fetch-and-delete)
   • Membership/ACL mutations (never eventually consistent)

FAILURE MODES TO NAME UNPROMPTED
─────────────────────────────────
✓ Reconnect storm after regional outage → backoff+jitter, admission control
✓ Partial/out-of-order fanout → client-side dedup + reorder by seq_no
✓ Push notification storms → coalesce, per-platform rate limits
✓ Large-group fanout & read-receipt amplification → hybrid strategy + batching
✓ Prekey exhaustion/replay → atomic consumption, pool monitoring
✓ Stale ACL after removal → push-invalidated permission cache
✓ Clock skew → never order by wall-clock, only conversation_seq

STAFF-LEVEL SIGNALS TO HIT
───────────────────────────
✓ Name the E2EE-vs-compliance anchor constraint explicitly
✓ Separate message_id (dedup) from conversation_seq (ordering)
✓ Recognize Kafka partition leadership as a ready-made CP sequencer
✓ Hybrid fanout strategy by group size, not one-size-fits-all
✓ Per-device (not per-user) multi-device modeling under E2EE
✓ Two named CP islands inside an otherwise AP system, each justified
✓ Calls treated as a separate signaling + media subsystem
✓ Proactive failure-mode enumeration before being asked
```

---

*Guide compiled for Senior & Staff-level system design interview preparation.*
*Topics: Messaging Systems, WhatsApp, MS Teams, End-to-End Encryption, Fanout, Ordering, CAP/PACELC.*
*Cross-references: Kafka, Twitter Fan-out Architecture, Distributed Unique ID Generation, Distributed Key-Value Store (Dynamo-style), Notification System, File Upload & Malware Scanning, Proximity Service, CAP/PACELC Theorem.*
