# Dropbox — System Design Interview Guide
### Senior & Staff Engineer Level

---

## Table of Contents

1. [Problem Statement & Scope](#1-problem-statement--scope)
2. [Requirements](#2-requirements)
3. [Capacity Estimation](#3-capacity-estimation)
4. [Core Entities](#4-core-entities)
5. [APIs](#5-apis)
6. [High-Level Architecture](#6-high-level-architecture)
7. [Component 1: Client (Watcher, Chunker, Indexer)](#7-component-1-client-watcher-chunker-indexer)
8. [Component 2: Chunking Strategy](#8-component-2-chunking-strategy)
9. [Component 3: Block Storage (S3) Service](#9-component-3-block-storage-service)
10. [Component 4: Metadata Service & Metadata DB](#10-component-4-metadata-service--metadata-db)
11. [Component 5: Synchronization & Notification Service](#11-component-5-synchronization--notification-service)
    - [Local → Remote and Remote → Local, Step by Step](#local--remote-and-remote--local-step-by-step)
12. [Component 6: Upload & Download Flow](#12-component-6-upload--download-flow)
13. [Component 7: Deduplication (and the Encryption Tension)](#13-component-7-deduplication-and-the-encryption-tension)
14. [Component 8: Conflict Resolution & Versioning](#14-component-8-conflict-resolution--versioning)
15. [Component 9: Sharing & Permissions](#15-component-9-sharing--permissions)
16. [CAP / PACELC Positioning](#16-cap--pacelc-positioning)
17. [Failure Modes & Mitigations](#17-failure-modes--mitigations)
18. [Scalability & Sharding](#18-scalability--sharding)
19. [Deep Dive: Control Plane vs Data Plane Split](#19-deep-dive-control-plane-vs-data-plane-split)
20. [Senior vs Staff Answer Differentiators](#20-senior-vs-staff-answer-differentiators)
21. [Interview Time Allocation](#21-interview-time-allocation)
22. [Quick-Reference Cheatsheet](#22-quick-reference-cheatsheet)

---

## 1. Problem Statement & Scope

**Dropbox** (and equivalently Google Drive, OneDrive, Box) is a **cloud file storage and synchronization system**: a user's files live in a local folder on N devices, and any change on one device — create, edit, delete, rename, move — propagates to every other device and to a durable cloud copy, without requiring the user to think about it.

The problem is really two problems wearing one trenchcoat:

1. **A blob storage system** — durably store arbitrary-sized files at scale, cheaply, with high availability.
2. **A distributed synchronization / replication problem** — keep N independent, possibly-offline replicas of a mutable file tree converging toward the same state, resolving conflicts sanely when they diverge.

### What the interviewer is really testing

- Can you cleanly separate **control plane (metadata)** from **data plane (bytes)** — a decision with CAP consequences that ripples through the whole design?
- Do you understand **chunking, delta sync, and deduplication** as the mechanisms that make "sync my file" tractable at scale, instead of "re-upload the whole file every time"?
- Can you reason about **conflicting concurrent writes** from multiple devices — including offline devices — without hand-waving?
- Do you make **explicit CAP/PACELC decisions per component**, the same way you would for any other distributed system?

---

## 2. Requirements

### Functional Requirements

| Requirement | Notes |
|---|---|
| Upload / download files | Core operation; must support arbitrarily large files |
| Sync across multiple devices | Changes on device A appear on devices B, C... |
| Offline editing | Client must queue changes and reconcile on reconnect |
| Version history | Restore a prior version of a file |
| File/folder sharing | Share with other users, with permission levels |
| Rename / move files & folders | Should be a metadata-only operation, not a re-upload |
| Delete / restore (trash) | Soft-delete with retention window |
| Efficient re-sync of edited files | Upload only the changed portion, not the whole file (delta sync) |

### Non-Functional Requirements

| Requirement | Target |
|---|---|
| Scale | ~500M registered users, ~100M DAU |
| Durability | 11 nines (no data loss) — this is non-negotiable, unlike availability |
| Availability | 99.9%+ for metadata/API; brief unavailability tolerable, data loss is not |
| Latency | Small file sync propagation < few seconds to online devices |
| Storage efficiency | Global block-level dedup; delta sync minimizes bandwidth |
| Consistency | Strong consistency on a single file's version chain; eventual consistency on cross-device propagation |
| Multi-device | A user may have 10+ simultaneously syncing devices |

**Staff-level framing up front:** durability and availability are **not the same requirement** here, and they pull the design in different directions. You can be briefly unavailable and lose no trust. You cannot silently lose or corrupt a byte of a user's file and keep their trust. State this distinction early — it sets up the CAP table later.

---

## 3. Capacity Estimation

*(Treat all figures below as reasoning tools to demonstrate estimation method, not pinned facts to defend under pressure.)*

```
Assumptions:
  500M registered users, 100M DAU
  Avg storage consumed per user: 5 GB (blended free/paid tiers)
  Avg chunk size: 4 MB

Total logical storage (pre-dedup):
  500M × 5GB = 2.5 EB (exabytes)

Global block-level dedup (illustrative 25% reduction from
common files: OS installers, popular PDFs, shared media):
  Physical storage ≈ 1.9 EB

Daily sync activity:
  Assume 10% of DAU perform ≥1 sync action/day → 10M sync events/day
  Avg 3 file versions touched per sync event → 30M file-version writes/day

Chunk churn from those writes:
  Avg 3 chunks touched per file version (delta sync — not the whole file)
  → 90M chunk uploads/day
  → 90,000,000 / 86,400 ≈ 1,042 chunks/sec average
  → peak (3× headroom) ≈ 3,125 chunks/sec

Ingest bandwidth:
  1,042 chunks/sec × 4MB ≈ 4.1 GB/s average
  Peak ≈ 12 GB/s

Metadata write QPS:
  30M file-version writes/day ≈ 347/sec average, ~1,000/sec peak
  → trivial for a well-sharded relational or wide-column store

Metadata DB size:
  Assume 500M users × 200 files avg × ~1KB metadata row (incl. version history)
  ≈ 100M... → 500M × 200 × 1KB = 100 TB of metadata
  → far larger than any single node; sharding is mandatory, not optional
```

**Key insight (mirrors the crawler guide's storage insight):** the *bytes* dominate total storage cost, but the *metadata* dominates system complexity. A 4MB chunk write to an object store is a solved problem. Correctly updating a sharded metadata store's version chain, under concurrent writers, without losing an update — that's where the design gets interesting.

---

## 4. Core Entities

Naming these up front separates the two problems from [Section 1](#1-problem-statement--scope) cleanly: entities that describe **bytes** (Chunk, Block) live in the data plane; entities that describe **structure and history** (File, Version, Namespace) live in the control plane. Conflating the two is where ad-hoc designs get into trouble.

| Entity | Description | Key Fields |
|---|---|---|
| **User** | A registered account; the root of ownership for personal namespaces | `user_id, email, storage_quota_bytes` |
| **Device** | One client installation (desktop, mobile, web) belonging to a user, with its own local chunk cache | `device_id, user_id` |
| **Namespace** | A file tree root — either a user's personal root or a shared folder; the unit sharing and permissions attach to | `namespace_id, owner_id, is_shared, tree_root` |
| **File** | A path within a namespace and a pointer to its current version | `file_id, namespace_id, path, current_version, is_deleted` |
| **Version** | An immutable snapshot of a file's content, identified by an ordered list of chunk hashes | `file_id, version_number, chunk_hash_list[], size_bytes, modified_by_device_id, created_at, is_conflict_copy` |
| **Chunk** | A content-defined, immutable slice of file bytes, addressed by its own hash — the unit of dedup, delta sync, and block storage ([Section 8](#8-component-2-chunking-strategy)) | `chunk_hash (SHA-256), size_bytes, ref_count, status (uploading\|uploaded)` |
| **Block** | The physical object-store record for one Chunk's bytes | `chunk_hash, storage_location` |
| **NamespaceMember** | A user's role within a shared Namespace | `namespace_id, user_id, role (owner\|editor\|viewer), joined_at` |
| **DeviceConnection** | A Device's durable sync state for one Namespace — one row per `(device_id, namespace_id)` pair, surviving disconnects; also doubles as the live-socket record for notification fan-out ([Section 11](#11-component-5-synchronization--notification-service)) while online | `device_id, namespace_id, last_synced_version_cursor, connection_id (null while offline)` |

```
User ──1:N── Device
User ──1:N── Namespace (personal root) ──N:M── User (via NamespaceMember, for shared namespaces)
Namespace ──1:N── File ──1:N── Version (ordered, immutable, append-only)
Version ──N:M── Chunk (ordered hash list)  ──1:1── Block (physical bytes)
Device ──N:M── Namespace (via DeviceConnection — one durable row per pair,
                          each tracking that pair's own sync cursor
                          independently of every other namespace)
```

**Why the cursor can't live on Device:** a device syncs multiple namespaces independently (`Device ──N:M── Namespace`), and `GET /namespaces/{id}/changes` takes its `since` cursor scoped *per namespace_id* ([Section 5](#5-apis), #5) — a single scalar on Device would have namespace A's sync progress clobber namespace B's. DeviceConnection is already the N:M join row for this pair, so that's where the per-namespace cursor has to live — it just has to be durable (outlive the socket) rather than deleted on disconnect, since reconnect logic resumes from "my last cursor for this namespace," not "since I reconnected" ([Section 11](#11-component-5-synchronization--notification-service)).

---

## 5. APIs

### Client-Facing API

Resource nouns map directly onto the Core Entities from [§4](#4-core-entities): `files`, `chunks`, `namespaces`, `devices`. The two CAS-gated writes (commit, delete) use the standard HTTP conditional-write mechanism — `If-Match: {expected_version}` — instead of a hand-rolled `expected_version` body field; a mismatch is a real `412 Precondition Failed`, not an app-level status string.

| S.No | Endpoint | Headers | Request Body | Response | Notes |
|---|---|---|---|---|---|
| 1 | `GET /chunks/{hash}` | — | — | `200 OK` (exists) / `404 Not Found` | Dedup check, called before uploading each chunk ([Section 12](#12-component-6-upload--download-flow)); existence conveyed by status code alone, no body needed; avoids re-uploading bytes already in Block Storage (S3) |
| 2 | `POST /chunks/{hash}/upload-url` | — | `size_bytes` | `presigned_url`, `expires_at` | Only called for chunks `GET /chunks/{hash}` reported missing (404). Chunk Broker writes the chunk record with `status=uploading` *before* returning, then hands back a time-limited, direct-to-object-store PUT URL — client uploads bytes straight to Block Storage (S3), Chunk Broker never sees them |
| 3 | `POST /files/{id}/versions` | — | `namespace_id`, `path` | `201 Created`: `version_token`, `expected_current_version` | Reserves the CAS baseline for the commit below — a pending Version resource |
| 4 | `PATCH /files/{id}` | `If-Match: {expected_version}` | `new_version: int`, `chunk_hash_list[]`, `size_bytes`, `device_id` | `200 OK`: `version_number` / `412 Precondition Failed` / `409 Conflict` | CAS on `current_version` ([Section 10](#10-component-4-metadata-service--metadata-db)) — `If-Match` mismatch is a real `412`, routed to conflict resolution ([Section 14](#14-component-8-conflict-resolution--versioning)); `409 Conflict` means a referenced chunk's S3 upload notification hasn't landed yet — client retries after a short backoff ([Section 12](#12-component-6-upload--download-flow)) |
| 5 | `GET /namespaces/{id}/changes?since={cursor}&limit={int, default 500, max 1000}` | — | — | `changed_files: [{file_id, path, current_version, is_deleted}]`, `new_cursor`, `has_more: bool` | Pulled by a client after a sync ping, or on reconnect. **Paginated, not a one-shot dump:** a namespace with heavy activity (bulk import, large shared folder) can have far more changed files than fit in one response. `limit` caps the page size; `has_more=true` means the client loops — call again with `since={new_cursor}` — until `has_more=false`, at which point it's fully caught up ([Section 11](#11-component-5-synchronization--notification-service)) |
| 6 | `GET /files/{id}/versions/{version_number}` | — | — | `chunk_hash_list[]`, `size_bytes`, `is_conflict_copy` | `version_number` path segment is optional — omit it (`GET /files/{id}/versions/latest`) for the current version. Client diffs the hash list against its local chunk cache before downloading |
| 7 | `GET /chunks/{hash}/download-url` | — | — | `presigned_url` | Returns a time-limited, direct URL rather than proxying bytes through the Chunk Broker — straight to Block Storage (S3) for most (private, cold) chunks, or to a CDN edge for "hot"/shared chunks ([Section 9](#9-component-3-block-storage-service)) |
| 8 | `DELETE /files/{id}` | `If-Match: {expected_version}` | — | `204 No Content` / `412 Precondition Failed` | Soft-delete (sets `is_deleted`, retained for the trash window); `If-Match` mismatch is `412`, same CAS semantics as the commit above |
| 9 | `GET /files/{id}/versions` | — | — | `versions: [{version_number, created_at, device_id, is_conflict_copy}]` | Powers version-history restore |
| 10 | `POST /files/{id}/versions/{version_number}/restore` | `If-Match: {expected_current_version}` | — | `200 OK`: `new_version_number` / `412 Precondition Failed` | Implemented as a new commit (`PATCH /files/{id}`) pointing at an old chunk list — history is never mutated |
| 11 | `POST /namespaces/{id}/members` | — | `target_user_id`, `role: owner\|editor\|viewer` | `201 Created` / `4xx` on failure | Creates a NamespaceMember row; enforced at the Metadata Service on every subsequent access ([Section 15](#15-component-9-sharing--permissions)) |
| 12 | `POST /devices/{id}/subscriptions` | — | `namespace_ids[]` | `connection_id` | Opens the long-lived WebSocket/long-poll channel; upserts one durable `DeviceConnection` row per `(device_id, namespace_id)`, attaching the live `connection_id` to each without disturbing that row's `last_synced_version_cursor` |

**Why reserve and commit didn't become one endpoint:** REST noun conventions rename the *shape* of these two calls (`POST .../versions` to reserve, `PATCH /files/{id}` to commit) but don't collapse them into one — they're still two round trips because the chunk-upload loop has to happen *between* them ([§6](#6-high-level-architecture)'s data-plane diagrams). What changes is that the second call's concurrency check now rides a standard HTTP header instead of an app-defined field, so a generic HTTP client/proxy/cache can reason about it without knowing this API's conventions.

### Internal (Service-to-Service) APIs

Only one of these four is a genuine synchronous request/response call — the other three are **messages** (Kafka produce/consume, a raw WebSocket push, an AWS-delivered queue event), not resources being fetched or mutated. Forcing a REST noun+verb shape onto a fire-and-forget event is the wrong move: there's no resource to `GET`/`PATCH`, just a fact being broadcast. The idiomatic convention for those is event-naming — `<resource>.<past-tense-verb>` — not an HTTP path at all; the "Endpoint" column below is replaced by "Topic / Channel" for exactly that reason. `ResolvePermission` is the one that *is* a real internal RPC against a resource, so it gets the same REST-noun treatment as the client-facing table.

| S.No | Transport | Topic / Channel / Endpoint | Caller → Callee | Payload | Response | Used in |
|---|---|---|---|---|---|---|
| 1 | Kafka (produce) | Topic `file-version-events`, event `file.version.committed` | Metadata Service → Notification Service | `namespace_id, file_id, new_version` | — (fire-and-forget to topic) | Sync & Notification Service ([Section 11](#11-component-5-synchronization--notification-service)) |
| 2 | WebSocket / long-poll push | Message type `namespace.changed` | Notification Service → Device (over open socket) | `namespace_id, latest_known_version` | `ack` (socket-level) | Tells the device to call `GET /namespaces/{id}/changes`, never carries file bytes |
| 3 | HTTP (internal RPC) | `GET /namespaces/{id}/members/{user_id}` | Metadata Service → NamespaceMember store | — (path params only) | `role: owner\|editor\|viewer\|none` | Checked on every metadata read/write against a shared namespace |
| 4 | AWS S3 Event Notification (via SQS/SNS) | Event `s3:ObjectCreated:Put` (AWS-defined name — not ours to rename) | Block Storage (S3) → Chunk Broker | `bucket, object_key (= chunk_hash), size_bytes, etag` | — (fire-and-forget; Chunk Broker consumes the queue) | Authoritative upload confirmation — the **only** thing that flips a chunk's `status` from `uploading` to `uploaded` and commits `ref_count`. Not driven by the client, so it's correct even if the client crashes right after the PUT ([Section 12](#12-component-6-upload--download-flow)) |

---

## 6. High-Level Architecture

```
                 ┌────────────────────────────────────────────────┐
                 │   Client                                         │
                 │                                                   │
                 │  ┌───────────────────┐   ┌─────────────────────┐ │
                 │  │ Uploader           │   │ Downloader           │ │
                 │  │ Watcher → Chunker   │   │ WS/long-poll         │ │
                 │  │ → Compress/Encrypt  │   │ listener →           │ │
                 │  │ → push local        │   │ GET .../changes      │ │
                 │  │   changes out       │   │ → GET .../chunks     │ │
                 │  └──────────┬──────────┘   └──────────┬───────────┘ │
                 │             └──────── Indexer / Local SQLite ───────┘ │
                 │                 (shared sync-state source of truth)   │
                 └───────────────────────────┬───────────────────────────┘
                              │  HTTPS (Uploader)        ▲  WS/long-poll (Downloader,
                              │                          │  push notifications land here)
                         ┌────────────▼───────────────┐
                         │   API Gateway / LB          │
                         └───────┬──────────┬──────────┘
                                 │          │
                 ┌───────────────▼──┐   ┌───▼──────────────────────┐
                 │  Metadata Service │   │  Chunk Broker             │
                 │  (file tree,      │   │  dedup check · issues     │
                 │   versions, ACLs) │   │  presigned URLs · chunk   │
                 └───┬───────────┬───┘   │  commit (bytes never      │
                     │           │       │  touch this service)      │
                     │           │       └──────────┬────────────────┘
                     │           │          issues a presigned URL,
                     │           │          flavor depends on chunk:
                     │           │       ┌───────────┴────────────┐
                     │           │       ▼                        ▼
                     │           │  hot/shared chunk        cold/private chunk,
                     │           │  (download only)          + all uploads
                     │           │  ┌──────────────┐        ┌──────────────────┐
                     │           │  │ CDN            │──────►│ Block Storage     │
                     │           │  │ (e.g. Cloud-   │ origin│ (S3)              │
                     │           │  │  Front edge)   │ fetch │                   │
                     │           │  │ cache miss →   │ on    │                   │
                     │           │  │ pull from S3;  │ miss  │                   │
                     │           │  │ cache hit →    │       │                   │
                     │           │  │ serve from edge│       │                   │
                     │           │  └──────────────┘        └──────────────────┘
         ┌───────────▼──┐   ┌────▼─────┐
         │ Metadata DB   │   │ Metadata │          (see presigned-URL
         │ (sharded,     │   │ Cache    │           data path below)
         │  e.g. by      │   │ (Redis)  │
         │  user_id/     │   └──────────┘
         │  file_id)     │
         └───────┬───────┘
                 │  change events
     ┌───────────▼───────────────┐
     │   Message Queue (Kafka)   │   ← durable, replayable, decouples
     └───────────┬───────────────┘     write path from fan-out path
                 │
     ┌───────────▼────────────────┐
     │  Synchronization /          │
     │  Notification Service       │
     │  (pub-sub, long-poll / WS)  │
     └───────────┬──────────────────┘
                 │
       fan-out "namespace changed" event
       to every online client subscribed
       to that folder/namespace
       — delivered into the Downloader side
       of whichever client instance that is
```

**The one-line architectural summary:** metadata flows through a **strongly-consistent, sharded control plane**; bytes flow through a **content-addressed, highly-available data plane**, with a CDN edge in front of Block Storage (S3) for the minority of chunks that are "hot" (shared/popular) rather than every private download; and a **durable pub-sub layer** bridges the two to tell other devices "something changed, go pull the new metadata." This split is elaborated in Section 19. (CDN placement and why it's opt-in per chunk, not default, is covered in [Section 9](#9-component-3-block-storage-service).)

**Uploader vs. Downloader — why the Client box is drawn as two roles:** "the client" is really two logically distinct responsibilities that happen to usually run in the same process:

- **Uploader** — detects local changes and pushes them out: Watcher → Chunker → Compress/Encrypt → `POST /files/{id}/versions` / `PATCH /files/{id}` against the Metadata Service and chunk upload against the Chunk Broker. This is entirely the **Local → Remote push path** ([§11](#11-component-5-synchronization--notification-service)).
- **Downloader** — determines that a file changed *on the server* and pulls it down: WS/long-poll listener → `GET /namespaces/{id}/changes` → diff against local chunks → `GET /chunks/{hash}/download-url` → reassemble. This is entirely the **Remote → Local pull path** ([§11](#11-component-5-synchronization--notification-service)).

They share one thing — the Indexer / local SQLite index — because both need the same answer to "what version of this file do I have locally," just from opposite directions (one is about to make it stale on the server, the other is about to catch it up). On a typical desktop client both roles live in the same process and the split is purely logical. But the separation is real, not just diagrammatic: a mobile client opened in read-only/view mode, or a web-only "preview" client that never edits files, is a Downloader with no Uploader at all — and a headless ingestion worker that only pushes files in (e.g. a backup agent) is the reverse. [Section 7](#7-component-1-client-watcher-chunker-indexer) covers the Uploader's sub-components in detail; the Downloader's mechanics are the Remote → Local walk in §11.

### The data plane, zoomed in: presigned URLs (both directions)

The Chunk Broker box above is deliberately thin — it brokers a **hash and a time-limited URL**, but the actual bytes for *both* upload and download skip it entirely and flow directly between the client and Block Storage (S3). Proxying every chunk's bytes through the Chunk Broker would make it the bandwidth bottleneck it's specifically designed not to be.

`POST /files/{id}/versions`/`PATCH /files/{id}` (Section 5, #3 & #4) are out of scope for the diagram below — they're control-plane calls against the **Metadata Service**, not the Chunk Broker, and bookend the chunk-upload loop rather than appearing inside it: `POST /files/{id}/versions` once before any chunk uploads to reserve the CAS baseline, then the loop below runs per changed chunk, then `PATCH /files/{id}` once after all chunks confirm `status=uploaded` to CAS-write the new `chunk_hash_list[]` ([Section 12](#12-component-6-upload--download-flow)).

```
 Client                 Chunk Broker              Block Storage (S3)
   │                          │                             │
   │  1. GET /chunks/{hash}   │                             │
   │     → POST /chunks/{hash}/upload-url                   │
   │     (Section 5, #1 & #2) │                             │
   ├─────────────────────────►│                             │
   │                          │  writes chunk record,       │
   │                          │  status = "uploading"       │
   │                          │                             │
   │  2. presigned PUT url    │                             │
   │◄─────────────────────────┤                             │
   │                          │                             │
   │  3. PUT chunk bytes (direct — Chunk Broker not in hop) │
   ├──────────────────────────────────────────────────────►│
   │                          │                             │
   │                          │   4. S3 Event Notification  │
   │                          │◄────────────────────────────┤
   │                          │   (object_key = chunk_hash)  │
   │                          │                             │
   │                          │  5. verify + commit record, │
   │                          │     status = "uploaded",    │
   │                          │     ref_count += 1          │
   │                          │                             │

 Download is the mirror image, with a URL-flavor branch — for
 cold/private chunks, GET /chunks/{hash}/download-url (Section 5, #7)
 returns a presigned GET url and the client fetches bytes direct from
 Block Storage (S3), same as above. For hot/shared chunks (Section 9's
 CDN subsection), the Chunk Broker instead returns a presigned CDN URL:

   Client            CDN Edge (CloudFront)         Block Storage (S3)
     │  GET /chunks/{hash}/   │                            │
     │     download-url       │                            │
     │  → presigned CDN url   │                            │
     │───────────────────────►│                            │
     │                        │  cache MISS (first         │
     │                        │  request for this hash)    │
     │                        │  → pulls object from S3 ──►│
     │                        │◄──────── bytes ─────────────┤
     │                        │  caches at the edge         │
     │◄── bytes (from edge) ──┤                            │
     │                        │                            │
     │  (later request, same chunk, nearby user)            │
     │───────────────────────►│                            │
     │                        │  cache HIT — served         │
     │                        │  straight from edge,        │
     │                        │  S3 never touched           │
     │◄── bytes (from edge) ──┤                            │

 The user always downloads from the nearest CDN edge location,
 never from the S3 region directly, once a chunk is cached.
 Only chunks with status="uploaded" are ever handed out this way.
```

The confirmation step is **infra-pushed, not client-pushed**: S3 itself notifies the Chunk Broker once the object exists, via an S3 Event Notification delivered to an SQS queue (or SNS topic) the Chunk Broker consumes. This is the step that actually flips `status: uploading → uploaded` and commits `ref_count`. Deliberately *not* a client callback — if the client crashes the instant after its PUT succeeds, the object is still in S3 and the notification still fires, so the chunk still gets committed. A client-initiated "I'm done" call would silently drop that case (bytes landed, record never created) and needs its own reconciliation job to catch the orphan; the S3-notification path makes that failure mode structurally unreachable.

Why bytes are already compressed + encrypted *before* step 3: chunking, compression, and encryption all happen client-side (Section 7) — the Chunk Broker never holds plaintext, and convergent-encryption keys ([Section 13](#13-component-7-deduplication-and-the-encryption-tension)) are derived from the plaintext chunk the client already has locally, not from anything the server computes.

### The control-plane bookend: Metadata Service (`POST /files/{id}/versions` / `PATCH /files/{id}`)

Same "zoomed in" treatment, but for the other side of the hop described above: no presigned URL here, no S3 — this is a direct, synchronous call from the client to the **Metadata Service**, which owns the strongly-consistent CAS write on `current_version`.

```
 Client                 Metadata Service              Metadata DB
   │                          │                             │
   │  1. POST                 │                             │
   │     /files/{id}/versions │                             │
   │     (Section 5, #3)      │                             │
   ├─────────────────────────►│                             │
   │                          │  read current_version,      │
   │                          │  mint version_token          │
   │                          ├────────────────────────────►│
   │                          │◄────────────────────────────┤
   │                          │                             │
   │  2. version_token,       │                             │
   │     expected_current_version                            │
   │◄─────────────────────────┤                             │
   │                          │                             │
   │   ⋯ chunk-upload loop against the Chunk Broker (the    │
   │     diagram above) runs here, zero or more times ⋯     │
   │                          │                             │
   │  3. PATCH /files/{id}    │                             │
   │     If-Match: expected_version                          │
   │     body: new_version,    │                             │
   │     chunk_hash_list[]     │                             │
   │     (Section 5, #4)      │                             │
   ├─────────────────────────►│                             │
   │                          │  CAS: current_version ==    │
   │                          │  If-Match value?              │
   │                          ├────────────────────────────►│
   │                          │                             │
   │                          │  4a. match + every chunk    │
   │                          │  status="uploaded"            │
   │                          │  → write new version row,     │
   │                          │  bump current_version          │
   │                          │◄────────────────────────────┤
   │                          │                             │
   │                          │  4b. mismatch → no write,    │
   │                          │  412 Precondition Failed      │
   │                          │  (Section 14 resolves)        │
   │                          │                             │
   │                          │  4c. a listed chunk still    │
   │                          │  status="uploading" → no      │
   │                          │  write, 409 Conflict           │
   │                          │  (client retries, backoff)    │
   │                          │                             │
   │  5. 200 OK: version_number│                             │
   │     / 412 / 409          │                             │
   │◄─────────────────────────┤                             │
   │                          │                             │
   │                          │  6. on 200 OK only:            │
   │                          │  produce file.version.committed│
   │                          │  → Kafka → Notification Service│
   │                          │  (fan-out, Section 11)        │
```

Two things worth calling out against the Chunk Broker diagram above:

- **Who confirms what.** Chunk upload is confirmed by *S3*, infra-pushed and independent of client liveness (step 4/5 in the diagram above). Version commit is confirmed by the *Metadata Service itself*, synchronously, inside a single CAS write — there's no equivalent "notification" hop because there's no second party whose state needs reconciling.
- **Why the commit can still fail on healthy infra.** `409 Conflict` is the seam between the two diagrams: it's what the Metadata Service returns if the client races ahead and calls `PATCH /files/{id}` before every chunk in `chunk_hash_list[]` has actually flipped to `status="uploaded"` in the Chunk Broker's records — i.e., before the *other* diagram's step 5 has landed for all chunks. The client just retries after a short backoff rather than treating it as a real conflict (that's what `412 Precondition Failed` — a genuine `If-Match` mismatch — is for).

### Putting it all together: one change, every component, end to end

The two diagrams above are each one hop zoomed in. This stitches them into the full trip — Client A's edit on disk becoming bytes Client B can read — through every component in [§6](#6-high-level-architecture)'s architecture diagram: Metadata Service, Metadata DB, Chunk Broker, Block Storage (S3), Kafka, Notification Service, and the CDN edge. Numbering runs continuously across both halves so it reads as one trip; the prose walk this is the picture of lives in [§11](#local--remote-and-remote--local-step-by-step).

**Half 1 — Local → Remote: Client A's change reaches durable storage and gets published**

```
 Client A          Metadata Service      Chunk Broker        Block Storage (S3)     Kafka
 (uploader)        (control plane)       (data plane broker) (data plane bytes)    (durable topic)
    │                     │                     │                     │                  │
    │ 1. POST /files/{id}/versions                │                     │                  │
    ├────────────────────►│                      │                     │                  │
    │                     │ CAS-read current_version against Metadata  │                  │
    │                     │ DB (Cassandra/DynamoDB-style, sharded by   │                  │
    │                     │ user_id/file_id — §10)                     │                  │
    │ 2. version_token,   │                      │                     │                  │
    │    expected_version │                      │                     │                  │
    │◄────────────────────┤                      │                     │                  │
    │                     │                      │                     │                  │
    │ 3. GET /chunks/{hash} → POST /chunks/{hash}/upload-url, per missing chunk            │
    ├────────────────────────────────────────────►│                     │                  │
    │                     │                      │ writes chunk record,│                  │
    │                     │                      │ status="uploading"  │                  │
    │ 4. presigned PUT url│                      │                     │                  │
    │◄───────────────────────────────────────────┤                     │                  │
    │                     │                      │                     │                  │
    │ 5. PUT chunk bytes — direct, skips Metadata Service + Chunk Broker                   │
    ├─────────────────────────────────────────────────────────────────►│                  │
    │                     │                      │ 6. S3 Event          │                  │
    │                     │                      │    Notification      │                  │
    │                     │                      │◄─────────────────────┤                  │
    │                     │                      │ verify + commit,     │                  │
    │                     │                      │ status="uploaded"    │                  │
    │                     │                      │                     │                  │
    │ 7. PATCH /files/{id}  If-Match: expected_version  body: new_version, chunk_hash_list[]│
    ├────────────────────►│                      │                     │                  │
    │                     │ CAS-write current_version → new_version    │                  │
    │                     │ into the same Metadata DB as step 2         │                  │
    │ 8. 200 OK,          │                      │                     │                  │
    │    version_number   │                      │                     │                  │
    │◄────────────────────┤                      │                     │                  │
    │                     │ 9. produce file.version.committed{namespace_id, file_id, new_version}│
    │                     ├──────────────────────────────────────────────────────────────►│
```

**Half 2 — Remote → Local: Kafka fans the event out, Client B pulls the bytes it's missing**

```
 Kafka             Notification Svc      Client B              Chunk Broker       CDN / Block Storage (S3)
 (durable topic)   (consumer, WS fleet)  (downloader, online)  (data plane broker)
    │                      │                     │                     │                     │
    │ 10. consume event    │                     │                     │                     │
    ├─────────────────────►│                     │                     │                     │
    │                      │ 11. push "namespace changed" over open    │                     │
    │                      │     WS/long-poll socket — no bytes,       │                     │
    │                      │     no metadata, just "go look"           │                     │
    │                      ├────────────────────►│                     │                     │
    │                      │                     │                     │                     │
    │                      │                     │ 12. GET /namespaces/{id}/changes →         │
    │                      │                     │     Metadata Service → {changed_files[],   │
    │                      │                     │     new_cursor, has_more} — loops (page by  │
    │                      │                     │     page) if has_more=true until caught up  │
    │                      │                     │                     │                     │
    │                      │                     │ 13. GET /files/{id}/versions/{n} →         │
    │                      │                     │     Metadata Service → chunk_hash_list[]   │
    │                      │                     │                     │                     │
    │                      │                     │ 14. diff against local chunk cache —       │
    │                      │                     │     only request bytes actually missing    │
    │                      │                     │                     │                     │
    │                      │                     │ 15. GET /chunks/{hash}/download-url, per   │
    │                      │                     │     missing chunk  │                     │
    │                      │                     ├────────────────────►│                     │
    │                      │                     │                     │ cold/private →      │
    │                      │                     │                     │ presigned S3 GET url│
    │                      │                     │                     │ hot/shared →        │
    │                      │                     │                     │ presigned CDN URL   │
    │                      │                     │◄────────────────────┤                     │
    │                      │                     │                     │                     │
    │                      │                     │ 16. GET bytes — direct from S3 (cold), or  │
    │                      │                     │     via the CDN edge, which origin-fetches │
    │                      │                     │     from S3 only on a cache miss           │
    │                      │                     ├───────────────────────────────────────────►│
    │                      │                     │◄───────────────────────────────────────────┤
    │                      │                     │ 17. reassemble chunks → file on disk        │
```

**Reading the seam between the two halves:** everything touching bytes (steps 3-6, 15-16) is the **data plane**; everything touching version numbers (1-2, 7-9, 12-13) is the **control plane**; step 11 is the *only* thing handed directly from one plane's infrastructure to the other, and it deliberately carries zero bytes and zero metadata — just "something changed, go pull it" ([§11](#11-component-5-synchronization--notification-service)). That one thin, content-free handoff is why the two planes can scale, fail, and be reasoned about independently.

---

## 7. Component 1: Client (Watcher, Chunker, Indexer)

The client is not a thin uploader — it's a meaningful piece of the distributed system, responsible for detecting change, minimizing bytes sent, and surviving being offline.

This section covers the **Uploader** role specifically (see [§6](#6-high-level-architecture) for the Uploader/Downloader split) — it could be a web browser, mobile app, or desktop app, and it is the piece responsible for proactively detecting local changes and pushing them to remote storage. The **Downloader** role — determining that a file has changed on the remote server and pulling those changes down, which doesn't have to be the same client instance that uploaded them — is covered by the Remote → Local walk in [§11](#11-component-5-synchronization--notification-service).

### Sub-components

```
Watcher:
  • OS-level file system event hooks (inotify on Linux,
    FSEvents on macOS, ReadDirectoryChangesW on Windows)
  • Debounces rapid-fire events (e.g. an editor doing many
    small writes while saving) into a single "file changed" signal

Chunker:
  • Splits the changed file into content blocks (see Section 8)
  • Computes a hash per chunk (SHA-256) — this hash becomes
    the chunk's identity in block storage

Indexer:
  • Maintains the client's local state: which chunks make up
    which file, at which version, and their sync status
  • Local SQLite DB — survives client restarts
  • Reconciles local state against server metadata on reconnect

Local DB (SQLite):
  file_path → { file_id, version, chunk_hash_list[], sync_state }
```

### Why local state matters (Staff-level point)

A naive design treats the client as stateless — just diff the folder against the server on every sync. That doesn't scale: hashing every file in a user's Dropbox on every reconnect is wasted CPU and I/O for a laptop with 200GB of files.

**The local DB lets the client answer "what changed since I was last online?" without re-scanning everything** — the watcher's events plus the local DB's last-known-good state are sufficient. Full-tree reconciliation (rehash everything, diff against server) is the fallback path used only after detecting local DB corruption or an unclean shutdown, not the steady-state path.

---

## 8. Component 2: Chunking Strategy

This is where "sync a 2GB video file after changing one frame" either becomes a 4MB re-upload or a 2GB re-upload — arguably the single highest-leverage design decision in the whole system for bandwidth cost.

### Fixed-Size Chunking

```
Split file into fixed N-byte blocks (e.g. 4MB), boundaries at
byte offsets 0, 4MB, 8MB, 12MB...

Pro:  Simple; easy to compute; easy to parallelize upload
Con:  A single byte inserted at the start of the file shifts
      every subsequent chunk boundary → every chunk hash changes
      → delta sync degrades to a full re-upload for edits that
      insert/delete bytes (not just overwrite in place)
```

### Content-Defined Chunking (CDC) — Staff-level answer

```
Boundaries are determined by the CONTENT itself, not fixed offsets,
using a rolling hash (Rabin-Karp fingerprint) over a sliding window:

  Slide a window across the byte stream, computing a rolling hash.
  Declare a chunk boundary whenever the hash satisfies a condition,
  e.g.  hash(window) mod 4096 == 0
  → produces variable-size chunks, average size configurable via
    the modulus (e.g. average 4MB, min 2MB, max 8MB)

Why this fixes the insertion problem:
  Inserting a byte shifts the rolling hash locally, but the window
  "resyncs" to the same boundary pattern a few bytes later. Only
  the chunk(s) touching the edit change; every chunk before and
  after the edit is byte-identical and hash-identical to before.
```

**Senior-level answer:** "We'll chunk the file into fixed 4MB blocks."
**Staff-level answer:** "We'll use content-defined chunking with a rolling hash so that byte insertions/deletions only invalidate the chunks touching the edit — fixed-size chunking would force a full re-upload on any non-overwrite edit."

### Chunk Metadata

```
chunk_hash (SHA-256) → { size, ref_count, storage_location }

The chunk hash is the block's identity across the ENTIRE system —
this is what makes global deduplication possible (Section 13).
```

---

## 9. Component 3: Block Storage Service

### What it is

A **content-addressed object store** — chunks are stored keyed by their hash, not by filename or user. Two users' identical chunks (e.g. the same PDF template) are physically the same object.

```
Storage key:  s3://blocks/{hash[0:2]}/{hash[2:4]}/{full_hash}
              (prefix sharding avoids S3 key hot-spotting)

Properties:
  • Immutable — a chunk is never modified in place, only
    created (on first upload) or reference-counted to zero
    and garbage-collected
  • Content-addressed — write is naturally idempotent:
    uploading the same bytes twice is a no-op after the first
  • Replicated (RF=3 or erasure-coded) for durability
```

### Chunk Broker — the gatekeeper in front of storage

```
Client → Chunk Broker:
  1. Client sends chunk_hash (not the bytes yet)
  2. Chunk Broker checks: does this hash already exist in
     block storage? (dedup check — see Section 13)
  3a. Exists (status=uploaded) → respond "skip upload", ref_count += 1
  3b. Doesn't exist → Chunk Broker writes a chunk record with
        status="uploading", then responds with a presigned PUT url
        (time-limited, direct-to-object-store)
  4. Client compresses + encrypts locally, then PUTs bytes
     STRAIGHT TO OBJECT STORE — Chunk Broker is not in this hop
  5. S3 fires an Event Notification on object creation → delivered
     to the Chunk Broker (via SQS/SNS), NOT called by the client
  6. Chunk Broker verifies the notification (object_key == chunk_hash,
     size matches) and commits the chunk record: status="uploaded",
     ref_count = 1
  7. Client, having polled/been notified the chunk is "uploaded"
     (or simply proceeding optimistically — see Section 12), reports
     the chunk as synced to Metadata Service
```

This two-phase "ask before you send" protocol is what makes delta sync + global dedup actually save bandwidth — the client never uploads bytes the server already has. And because the actual bytes travel over a **presigned URL straight to the object store**, the Chunk Broker never becomes a bandwidth chokepoint on the write path either — it only ever brokers hashes and short-lived URLs, exactly as it already does for downloads. Confirmation is deliberately **S3-pushed, not client-pushed**: a client that crashes right after its PUT succeeds still gets its chunk committed, because S3's notification fires independently of client liveness. A client-ack design would need a separate reconciliation sweep to catch that exact crash window; this design doesn't need one.

### CDN / Edge Caching — for popular & shared chunks, not every private download

Direct-to-object-store presigned URLs ([Section 12](#12-component-6-upload--download-flow)) are the right default for Dropbox's actual traffic shape: most downloads are one user pulling their own private files to their own devices — there's no "many readers, one object" fan-out to amortize an edge cache against. A CDN only earns its cost on the minority of chunks that genuinely get re-requested from many locations: public share links, a chunk shared across a large team's Namespace, or a chunk that happens to be popular because it's deduplicated across many users' accounts (Section 13).

```
Approach:
  Client → Chunk Broker: GET /chunks/{hash}/download-url
  Chunk Broker decides URL flavor based on request signal
  (e.g. namespace.is_shared, or a hit-count threshold on that hash):
    - "hot" / shared chunk  → presigned CDN edge URL
    - "cold" / private chunk → presigned Block Storage (S3) URL (as today)
  Either way, it's a signed, time-limited URL — same security
  model as the upload side's presigned PUT, just GET instead

CDN populates its edge cache lazily on first miss, pulling the
chunk from Block Storage (S3) (CDN as a cache IN FRONT OF the object
store, not a separate copy we manage ourselves).
```

**Why cache invalidation — the usual hard part of CDN design — mostly disappears here:** chunks are immutable and content-addressed (Section 8). A chunk's bytes at hash `H` never change; "editing a file" produces a *new* chunk with a *new* hash, never a mutation of an existing cached object. There is no "stale CDN copy serving old content" failure mode the way there is for a mutable blob at a fixed URL. The only cache-management problem left is **cost**, not correctness:

```
Challenge: CDN cost scales with bytes cached × edge locations,
and most chunks are cold (one user, rarely re-fetched).

Mitigation:
  • Short default Cache-Control TTL (e.g. 24h) on chunk objects —
    rare chunks simply age out and re-fetch from Block Storage (S3)
    on the next (unlikely) request; no correctness cost since
    content never changes, only cache residency does
  • Only route through the CDN flavor of presigned URL for
    chunks flagged "hot" (shared namespace, or ref_count above
    a threshold) — private, low-ref-count chunks skip the CDN
    entirely and go straight to Block Storage (S3), same as before
  • On delete / ref_count → 0 (Section 17's orphan GC), there's
    no need to actively purge the CDN edge — the short TTL lets
    it age out naturally, and even if it didn't, the risk is
    "served one cached file slightly past its intended lifetime,"
    not "served wrong content"
```

---

## 10. Component 4: Metadata Service & Metadata DB

This is the **control plane** — and the component that most needs Staff-level rigor, because it's where correctness bugs (lost updates, phantom conflicts, orphaned versions) live.

### What Metadata Tracks

```
File record:
  file_id, owner_id, path, current_version, is_deleted,
  created_at, modified_at

Version record:
  file_id, version_number, chunk_hash_list[] (ordered),
  size, modified_by_device_id, modified_at, is_conflict_copy

Namespace/folder record:
  namespace_id (shared folder or personal root), owner_id,
  member_list[] (for sharing), tree structure
```

### Schema (Wide-Column, e.g. Cassandra/DynamoDB style)

```sql
CREATE TABLE file_metadata (
  user_id          TEXT,
  file_id          TEXT,
  path             TEXT,
  current_version  BIGINT,
  chunk_hashes     LIST<TEXT>,
  size_bytes       BIGINT,
  is_deleted       BOOLEAN,
  modified_at      TIMESTAMP,
  PRIMARY KEY (user_id, file_id)
);

CREATE TABLE file_versions (
  file_id          TEXT,
  version_number   BIGINT,
  chunk_hashes     LIST<TEXT>,
  device_id        TEXT,
  created_at       TIMESTAMP,
  is_conflict_copy BOOLEAN,
  PRIMARY KEY (file_id, version_number)
) WITH CLUSTERING ORDER BY (version_number DESC);
```

Partitioning by `user_id` collocates a user's whole file tree — cheap "list my files" queries — at the cost of potential hot partitions for power users with millions of files (mitigated with sub-partitioning by path prefix, see Section 18).

### The Write Path Needs a Single-Writer Guarantee Per File

Two devices editing the same file concurrently must not both win — the metadata write path needs **conditional writes** (compare-and-swap on `current_version`):

```
UPDATE file_metadata
SET current_version = new_version, chunk_hashes = ...
WHERE user_id = ? AND file_id = ? AND current_version = expected_version
IF current_version = expected_version   -- CAS / optimistic lock

If the CAS fails → another device already advanced the version →
this is a genuine conflict → route to conflict resolution (Section 14),
do NOT silently overwrite.
```

This CAS-on-version-number is the metadata-layer equivalent of the crawler guide's "distributed lock via Redis SET NX" — a narrow, cheap coordination point rather than a heavyweight distributed lock across the whole write path.

---

## 11. Component 5: Synchronization & Notification Service

### The Problem

When device A writes a new version, devices B and C (if online) need to find out — ideally within seconds, without every device polling the server every few seconds (which doesn't scale to 100M DAU).

### Design: Pub-Sub Fan-Out, Not Payload Push

```
Chunk Broker / Metadata Service, on successful commit:
  → publish {namespace_id, file_id, new_version} to Kafka topic

Notification Service (Kafka consumer):
  → looks up DeviceConnection rows for namespace_id with a live
    (non-null) connection_id — i.e. devices currently online
  → pushes a lightweight "namespace changed" ping to each
    (via long-lived WebSocket, or long-poll for constrained clients)

Client, on receiving the ping:
  → does NOT receive the file bytes in the push
  → calls Metadata Service: "what changed since my last known version
    for THIS namespace?" (since_version_cursor = this namespace's
    DeviceConnection.last_synced_version_cursor, not a device-wide value)
  → pulls only the metadata delta, then requests only the chunks
    it doesn't already have locally, and advances that same
    DeviceConnection row's cursor on success
```

**Why push a ping, not the payload:** the notification service's job is to say "go look," not to be the sync payload's transport. This keeps the pub-sub layer thin, stateless about content, and easy to scale horizontally — it never touches file bytes. See the **Kafka guide** and **Notification System guide** for the general pub-sub fan-out patterns this reuses.

### Long-Poll / WebSocket Connection Fleet

```
100M DAU, assume 20% concurrently online → 20M persistent connections

Connection servers: stateless-ish (hold the socket, not app state)
  ~50K connections/node → 400 nodes minimum, plan for 3× headroom

Each connection server subscribes to Kafka topics for the
namespaces its connected clients care about, OR (more commonly
at this scale) subscribes to a sharded internal pub-sub keyed by
namespace_id, fed by a Kafka consumer group.
```

### Thundering Herd on Reconnect (Staff-level gotcha)

If the notification fleet has an outage and recovers, or a mobile network flaps, **millions of clients reconnect simultaneously** and each immediately asks "what changed since my last version?" — a query storm against the Metadata DB.

**Mitigation:** jittered reconnect backoff on the client (don't reconnect at t=0 for everyone), and a metadata cache (Redis) in front of the DB for "latest version" lookups, since reconnect storms are read-heavy on a small hot set of recently-active namespaces.

### Local → Remote and Remote → Local, Step by Step

The pieces above (pub-sub fan-out, the connection fleet, reconnect handling) are the *notification* half of sync. The other half — how a change actually travels from one device's disk to another's — is split across the API table ([§5](#5-apis)), the Block Storage (S3) / chunk-upload mechanics ([§9](#9-component-3-block-storage-service)), and the Upload/Download flow ([§12](#12-component-6-upload--download-flow)). This section is the explicit, end-to-end walk connecting them, named by direction.

#### Local → Remote (push path): Device A's change reaches the server — the Uploader role

This is what happens when *this* device has new local state that needs to become the new source of truth; steps 1-2 are the Watcher/Chunker from [§7](#7-component-1-client-watcher-chunker-indexer), and the rest is the Uploader doing its half of the Uploader/Downloader split from [§6](#6-high-level-architecture). (Full chunk-level mechanics are in [§12](#12-component-6-upload--download-flow) — this is the version-commit skeleton around them.)

```
1. Detect:   Watcher sees a filesystem change, debounces rapid-fire
             edits into one signal (§7)
2. Chunk:    Chunker splits content, hashes each chunk (§8)
3. Reserve:  POST /files/{id}/versions (namespace_id, path)
             → expected_current_version (the CAS baseline)
4. Upload:   per missing chunk — GET /chunks/{hash}, POST
             /chunks/{hash}/upload-url, direct PUT to Block Storage
             (S3), S3 notification confirms status="uploaded" (§9, §12)
5. Commit:   PATCH /files/{id}  If-Match: expected_version
             body: new_version, chunk_hash_list[], ... — CAS on
             current_version; 412 (If-Match mismatch) → conflict-copy
             fork (§14), never silently overwritten
6. Fan out:  Metadata Service publishes {namespace_id, file_id,
             new_version} to Kafka — handoff into Remote → Local
7. Update:   the committing device also writes the new version into
             its OWN local SQLite index — it doesn't wait for a
             round-trip notification to learn about its own write
```

#### Remote → Local (pull path): Devices B and C find out and catch up — the Downloader role

This is what happens on every *other* device that needs to learn about A's change — including A's *own other devices* (same user, different machine), not just other users' devices in a shared namespace. This whole walk is the Downloader half of the Uploader/Downloader split from [§6](#6-high-level-architecture): it's the piece responsible for determining when a locally-held file has changed on the remote server and pulling that change down — and it doesn't have to be the same client that uploaded it. There are two cases, and the second is the one that actually guarantees correctness.

**Case 1 — B is online right now:**

```
1. Notification Service (Kafka consumer) looks up every device with
   a live DeviceConnection (via POST /devices/{id}/subscriptions) on
   this namespace_id
2. push message type `namespace.changed`(namespace_id,
   latest_known_version) over B's open socket — no file bytes, no
   metadata payload, just "go look"
3. B calls GET /namespaces/{id}/changes?since={cursor}&limit={n}
   → {changed_files: [{file_id, path, current_version, is_deleted}],
      new_cursor, has_more}
   — a single ping can represent thousands of changes (bulk import,
   large shared folder), so this is a loop: advance the cursor to
   new_cursor and re-call while has_more=true, only proceeding to
   step 4 once a page's changed_files is in hand
4. For each changed file in the current page: GET
   /files/{id}/versions/{version_number} → chunk_hash_list[]
5. B diffs that hash list against its OWN local chunk cache — chunks
   already present (prior version, or unrelated shared content) are
   skipped; dedup pays off on the READ side here, not just storage
6. GET /chunks/{hash}/download-url only for what's missing — presigned
   URL direct to Block Storage (S3), or a CDN edge for "hot" chunks
   (§9's CDN subsection) — never proxied through Chunk Broker
7. B reassembles the file from chunks in order, writes it to disk
8. B updates its local SQLite index, and the server advances the
   DeviceConnection(B, namespace_id).last_synced_version_cursor row
   (§4) — scoped to THIS namespace only, so B's other subscribed
   namespaces keep their own independent progress
```

**Protocol choice for step 2's "open socket":** the ping itself is tiny and one-directional (server → client, "go look"), but the transport carrying it has real options with real tradeoffs:

| Protocol | How it fits | Tradeoff |
|---|---|---|
| **WebSocket** ✅ preferred | Full-duplex, persistent TCP; client opens once, server pushes pings for the lifetime of the connection | Lowest latency and per-message overhead once connected; the default choice for desktop and web clients with stable connectivity |
| **HTTP long-polling** | Client issues an HTTP request the server holds open until there's a ping or a timeout, then immediately reopens it | Higher overhead (reconnect on every timeout) and slightly higher latency, but passes through corporate proxies/firewalls that block the WebSocket `Upgrade` handshake — the fallback, not the default |
| **Server-Sent Events (SSE)** | Simple unidirectional HTTP push, built-in client auto-reconnect | Fine for browser clients; less natively supported in desktop/mobile native socket stacks than a plain WebSocket, so it doesn't generalize across all client types the way WebSocket does |
| **MQTT (or similar lightweight pub-sub)** | Tiny message framing, built-in keep-alive and QoS levels | Good fit specifically for **mobile clients**, where battery/data overhead matters more than raw latency |
| **Platform push (APNs/FCM)** | Not a sync-channel protocol at all — wakes a suspended mobile app | **Necessary complement, not an alternative:** mobile OSes aggressively kill backgrounded sockets, so when the OS has suspended the app's WebSocket/MQTT connection, APNs/FCM is what wakes it. The app then reconnects and does the normal `GET /namespaces/{id}/changes` catch-up — functionally Case 2 (reconnect) triggered by an OS-level nudge instead of the user opening the app |

**Preferred stack:** WebSocket as the default persistent channel for desktop/web, with HTTP long-polling as the automatic fallback when the WebSocket upgrade is blocked, and platform push (APNs/FCM) layered on top for mobile to handle the case where the OS has already torn down the persistent connection. This mirrors §11's original framing ("long-lived WebSocket, or long-poll for constrained clients") — the addition here is naming mobile's OS-suspension case explicitly, since that's the scenario that actually dominates Case 2 traffic in practice, not just literal network outages.

**Case 2 — B was offline when the ping fired:**

A push is best-effort — if B's socket was dead, nothing re-sends the ping later. B catches up on reconnect instead, which is the actual correctness backstop this section's Thundering-Herd mitigation above is protecting:

```
1. B reconnects, re-issues POST /devices/{id}/subscriptions for its
   namespaces — this re-attaches a live connection_id to each
   DeviceConnection(B, *) row without touching that row's
   last_synced_version_cursor
2. B immediately calls GET /namespaces/{id}/changes, per namespace,
   using that namespace's LAST PERSISTED cursor (local SQLite copy of
   DeviceConnection(B, namespace_id).last_synced_version_cursor)
   — not "since I reconnected," since it may have been offline
   for hours or days, and other subscribed namespaces resume from
   their own independent cursor, not a single device-wide one
3. Identical to Case 1 steps 3-8 from here, including the page loop —
   a device back after days offline is exactly the case most likely
   to have thousands of changes pending, so pagination matters most
   here, not just on the live-ping path
```

This is also the designed fallback for any dropped/missed notification, not just reconnects — the notification is a hint to check, never the thing correctness depends on.

#### Both halves together

```
Device A                  Server (Metadata + Kafka)              Device B
   │                              │                                  │
   ├─ PATCH /files/{id} ───────────►│                                 │
   │                              ├─ publish change event            │
   │                              ├─ Notification Svc looks up ──────►│ (online: namespace.changed push)
   │                              │                                  ├─ GET /namespaces/{id}/changes
   │                              │                                  │   (loops page by page on has_more)
   │                              │◄─────────────────────────────────┤
   │                              ├─ changed_files + new_cursor,     │
   │                              │  has_more ───────────────────────►│
   │                              │                                  ├─ GET /files/{id}/versions/{n},
   │                              │                                  │   diff local chunks
   │                              │                                  ├─ GET /chunks/{hash}/download-url
   │                              │                                  │   (missing only, direct from
   │                              │                                  │   Block Storage (S3) / CDN)
   │                              │                                  ├─ write to disk, update local index;
   │                              │◄─────────────────────────────────┤   server advances DeviceConnection
   │                              │                                  │   (B, namespace_id).cursor to new_cursor
```

**The asymmetry worth naming explicitly:** Local → Remote is synchronous and CAS-gated — the commit either succeeds or conflicts, right now. Remote → Local is asynchronous and cursor-driven — the ping is just a hint to check; the actual guarantee is "replay from my last cursor," which holds whether the hint arrived, arrived late, or never arrived at all.

---

## 12. Component 6: Upload & Download Flow

### Upload (small-to-medium file)

```
1. Watcher detects change → Chunker splits into content-defined chunks
2. Client asks Metadata Service to start a new version, gets version token
3. For each chunk: ask Chunk Broker "do you have hash H?" (dedup check)
4. For each MISSING chunk: Chunk Broker writes a chunk record with
   status="uploading" and returns a presigned PUT url; client
   compresses + encrypts locally, then uploads the chunk DIRECTLY to
   Block Storage (S3) (parallelized, several chunks concurrently) — Block
   Server is never on the hook for the actual bytes
5. S3 fires an Event Notification per object created → Chunk Broker
   consumes it (SQS/SNS) and flips that chunk's status to "uploaded"
   + ref_count += 1. This is NOT a client call — it happens whether
   or not the client is still around to ask for it
6. Client commits the new version to Metadata Service with the
   ordered chunk hash list + CAS on version. `PATCH /files/{id}`
   validates every referenced chunk has status="uploaded" before
   accepting; if the S3 notification for a just-uploaded chunk
   hasn't landed yet (typically sub-second), it returns a retryable
   `409 Conflict` rather than committing a version that points at
   an unconfirmed chunk — client backs off briefly and retries
7. Metadata Service publishes change event → Notification Service fans out
```

### Upload (large file — multipart, resumable)

```
Large files are just "many chunks" — chunking already gives you
resumability for free: if the client crashes mid-upload, on restart
it re-asks the dedup-check step, which reports which chunks already
landed, and only the remaining chunks are sent. No separate
multipart-upload protocol is needed beyond what chunking already does.
```

### Download

```
1. Client metadata sync reveals a file at a version it doesn't have
2. Client fetches the version's chunk_hash_list from Metadata Service
3. Client diffs against its LOCAL chunk store — many chunks may
   already be present locally (e.g. from a prior version, or from
   another file that happened to share content)
4. Client downloads only missing chunks via `GET /chunks/{hash}/download-url` —
   Chunk Broker returns a pre-signed URL pointing at Block Storage (S3)
   for most (private, cold) chunks, or a CDN-signed URL for chunks
   flagged "hot" — shared-namespace or high ref_count (Section 9's
   CDN subsection). For the CDN branch: cache MISS (first request
   for that hash) → CDN pulls the object from Block Storage (S3) and
   caches it at the edge; cache HIT (any later request, from any
   user near that edge) → served straight from the edge, Block
   Storage never touched. Either way, bytes go DIRECT, not through
   the Chunk Broker, and the client ends up downloading from
   whichever edge location is nearest it rather than from the S3
   region directly
5. Client reassembles file from chunks in order, writes to disk
6. Client updates local DB with new version + chunk list
```

**Why pre-signed URLs matter:** routing every chunk's bytes through the Chunk Broker as a proxy — in *either* direction — makes it a bandwidth chokepoint sized to total upload+download throughput instead of just metadata QPS. Pre-signed, time-limited URLs let clients push/pull bytes directly to/from Block Storage (S3) (or, for hot chunks, a CDN edge in front of it — [Section 9](#9-component-3-block-storage-service)), and the Chunk Broker's job stays limited to authorization + metadata bookkeeping (dedup checks, issuing URLs, confirming commits) — never the bytes themselves. This is the same pattern used for upload ([Section 9](#9-component-3-block-storage-service), this section's upload flow above).

---

## 13. Component 7: Deduplication (and the Encryption Tension)

Two independent layers, same shape as the crawler guide's URL vs content dedup — but here the two layers are **chunk-level identity dedup** and a genuine architectural tension with **encryption**.

### Chunk-Level Dedup (mechanically simple)

```
Chunk identity = SHA-256(chunk_bytes)

If two users upload byte-identical chunks (a shared PDF, a common
OS file, a popular meme), the SECOND upload is a no-op after the
dedup check — same content, same hash, same storage object,
ref_count += 1.
```

### The Staff-Level Tension: Dedup vs Per-User Encryption

```
If each user's files are encrypted with a UNIQUE per-user key
before chunking/hashing:
  → identical plaintext produces DIFFERENT ciphertext per user
  → chunk hashes never match across users
  → global dedup is impossible; only WITHIN one user's own files
    (across their own versions/files) does dedup still work

If you want cross-user dedup AND encryption, the only lever is
CONVERGENT ENCRYPTION:
  encryption_key = hash(plaintext_chunk)
  → same plaintext always produces the same ciphertext,
    regardless of which user uploaded it
  → dedup works across users again

  Trade-off: convergent encryption is vulnerable to a
  "confirmation of file" attack — an attacker who already
  knows a candidate plaintext can check whether ANY user
  has that exact file, by computing the same hash and
  checking if the chunk exists server-side. This is a real,
  known weakness (relevant e.g. for copyrighted or illegal
  content detection — a double-edged property) and needs to
  be explicitly called out as an accepted trade-off, not
  glossed over.
```

**Senior-level answer:** "We dedup by hashing chunks."
**Staff-level answer:** "Cross-user dedup and per-user encryption are in tension. If we want both, convergent encryption is the standard approach, and it comes with a known confirmation-of-file side channel we'd need to explicitly accept or mitigate — for instance, only deduping within account boundaries, or only for a whitelisted class of non-sensitive file types."

---

## 14. Component 8: Conflict Resolution & Versioning

### When Conflicts Happen

```
Device A (offline) edits file.txt → local version 5
Device B (online) edits the same file.txt → server version advances
  from 4 to 5 via B
Device A reconnects, tries to commit ITS version 5
  → CAS against server's "current_version = 4" fails
    (server is already at 5, not 4)
  → genuine conflict: two divergent edits both claim to be
    the successor of version 4
```

### Resolution Strategy: Conflicted Copy (what Dropbox actually does)

```
On CAS failure at commit time:
  1. Do NOT silently overwrite the winning version
  2. Do NOT attempt automatic content merge (safe for structured
     text like code with CRDTs/OT, but Dropbox handles arbitrary
     binary files — merging a PSD or a zip file is not well-defined)
  3. Accept the LOSING device's version as a NEW file:
       "file (conflicted copy from Device A's name, date).txt"
  4. Both files now exist; the user resolves manually

This trades automatic correctness for USER-VISIBLE correctness —
the system never loses data (durability requirement), and never
guesses wrong about which edit the user "really wanted."
```

### Why Not Vector Clocks / CRDTs Here (a decision to defend, not a gap)

```
Vector clocks detect concurrent (non-causally-ordered) writes
precisely — useful when you WILL merge automatically (e.g. a
key-value store, per Dynamo). CRDTs go further and merge
automatically for specific data types.

For arbitrary binary files, there is no general-purpose merge
function. Detecting "these two versions diverged from a common
ancestor" (which a vector clock or version-number CAS both give
you) is necessary, but the harder question — merge or fork — has
only one safe general answer for opaque binary blobs: fork
(conflicted copy). Vector clocks would be over-engineering for a
signal you already get for free from the CAS version check;
they'd matter more if this were a live-collaboration document
editor (Google Docs-style), which is a different problem with
OT/CRDT-based merge at the character level.
```

This is a good place in an interview to proactively distinguish "sync a file" from "collaborative real-time editing" — conflating the two is a common Senior-level slip.

### Version History & Retention

```
Every committed version is retained (not overwritten) for a
retention window (e.g. 30 days standard, unlimited for
"Extended Version History" paid tier).

Old chunk versions are NOT deleted directly — deletion only
happens via reference counting: a chunk is garbage-collected
only when NO version of NO file anywhere still references it.
```

---

## 15. Component 9: Sharing & Permissions

### Shared Folder as a Namespace

```
namespace_id ≠ user_id — a shared folder is its own namespace with
its own member list and ACL, referenced by symlink-like entries in
each member's personal file tree.

namespace_members: { namespace_id, user_id, role (owner/editor/viewer),
                   joined_at }
```

### 60–90s Verbal Answer: "Users should be able to share a file with other users"

> For sharing, we model a shared folder as its own **Namespace** — completely separate from the sharer's personal namespace, with its own member list and its own ACL. When user A shares a folder with user B, we call `SHARE_NAMESPACE` with the namespace ID, B's user ID, and a role — owner, editor, or viewer. That writes one row into a **NamespaceMember** table: `namespace_id, user_id, role, joined_at`. That row is the single source of truth for who can access that namespace.
>
> To make it visible, we add a symlink-like reference entry into B's own personal file tree, so the shared folder just appears alongside their own files — no separate "shared with me" query required.
>
> Enforcement always happens server-side: every metadata read or write against that namespace checks the caller's role in NamespaceMember at the Metadata Service — we never trust the client to self-restrict. And for the actual bytes, we never hand out a durable raw link to storage; every chunk download is a presigned URL, scoped and time-limited to that one authorized request.
>
> One nuance worth calling out: this models sharing at *folder* granularity. Sharing a single file falls out of the same mechanism — it just means that file lives in a namespace scoped to itself.

### Fan-Out Cost of Sharing (Staff-level scaling concern)

```
A change in a folder shared with 500 people fans out to 500×
the notification load of a personal-folder change.

Mitigation, same principle as the Twitter fan-out guide's
celebrity problem:
  • Small shared folders (the overwhelming majority): fan-out on
    write, same as personal folders — push directly to all
    subscribed members.
  • Very large shared folders / "team spaces" (rare but real,
    e.g. a company-wide folder with 10K+ members): fan-out on
    READ instead — don't push individually to 10K sockets on
    every commit; instead bump a per-namespace version counter,
    and rely on clients' periodic/reconnect metadata pull to
    pick up the change, or batch-push at reduced frequency.

  This is the same hybrid fan-out-on-write / fan-out-on-read
  split covered in the Twitter / Fan-out Architecture guide,
  applied to file namespaces instead of social timelines.
```

### Permission Enforcement Point

```
ALWAYS enforce at the Metadata Service, not the client:
  • Every metadata read/write checks the caller's role in
    namespace_members for that namespace
  • Block Storage (S3) reads use pre-signed URLs scoped and
    time-limited per authorized request — never a durable,
    shareable direct link to raw storage
```

---

## 16. CAP / PACELC Positioning

Per-component, not system-wide — see the **CAP/PACELC Theorem guide** for the general framework this applies.

| Component | CAP Choice | PACELC (else) | Reasoning |
|---|---|---|---|
| Metadata DB (version write path) | **CP** | PC/EC | A file's version chain must never fork silently. CAS-on-version requires consistency; under partition, reject the write rather than risk two "current" versions. |
| Metadata DB (read path, e.g. "list my files") | **AP** (via cache) | PA/EL | Slightly stale file listing is acceptable; low latency and availability preferred for the common read path. |
| Block storage (chunks) | **AP** | PA/EL | Content-addressed and immutable — writes never conflict, so availability is free to prioritize. Durability (replication) is separate from the CAP availability axis. |
| Notification/pub-sub service | **AP** | PA/EL | Best-effort, at-least-once delivery. A missed ping is harmless — the client's periodic/reconnect metadata pull is the correctness backstop, not the notification. |
| Sharing/ACL store | **CP** | PC/EC | Permission checks must be consistent — briefly denying access during a partition is far safer than briefly over-granting it. |
| Local client DB (SQLite) | **AP** (trivially) | — | Single-node, always available to its own device; reconciles against server on reconnect. |

**The single most important CAP call in this system:** the metadata **write** path is CP and the metadata **read** path is AP-via-cache — these are the *same store*, deliberately given different consistency postures for different access patterns. This is the kind of per-operation (not just per-component) nuance that separates Staff from Senior answers.

---

## 17. Failure Modes & Mitigations

| Failure | Symptom | Detection | Mitigation |
|---|---|---|---|
| Client crash mid-upload | Partial chunk set uploaded, version never committed | Client's local DB shows "pending" chunks on restart | Chunking makes upload naturally resumable; re-run dedup-check step, only missing chunks re-sent; version never partially visible to other devices (commit is atomic). Any chunk whose bytes *did* land in S3 before the crash still gets committed via the S3 notification ([Section 9](#9-component-3-block-storage-service)) — the crash only orphans chunks that never reached S3 |
| S3 upload notification lost or delayed | Chunk stuck in `status="uploading"` indefinitely even though bytes exist in Block Storage (S3) | Periodic sweep for chunk records in `uploading` state past the presigned URL's `expires_at` | Reconciliation job does a `HEAD` on the object store for stale `uploading` chunks: object present → commit it (replays the missed notification's effect); object absent → the upload genuinely never happened, delete the stale record so the hash can be retried |
| Concurrent edits from two devices | CAS failure on version commit | Metadata Service returns version-mismatch error | Conflicted-copy fork (Section 14); never silently overwrite |
| Metadata DB partition | Risk of split-brain version forks | Write quorum failure / leader election events | CP posture on write path: reject writes on the minority side rather than risk divergent "current" versions |
| Notification service outage | Clients don't get real-time pings | Kafka consumer lag on notification topic; client heartbeat misses | Clients fall back to periodic full metadata poll (e.g. every few minutes) as backstop; reconnect triggers immediate metadata pull |
| Reconnect thundering herd | Metadata DB read spike after outage recovery | Sudden QPS spike correlated with notification-service recovery | Jittered client reconnect backoff; Redis cache absorbs the read spike |
| Chunk storage node failure | Risk of losing a chunk that many files reference | Storage replication health checks | Replication factor 3 (or erasure coding); a single lost replica is a non-event |
| Orphaned chunks after delete | Wasted storage cost | Periodic ref-count audit job | Reference counting; garbage collect only when ref_count hits 0 across ALL files/versions |
| Convergent-encryption confirmation attack | Attacker infers a user has a known file | N/A (architectural property, not an incident) | Explicitly scope cross-user dedup to non-sensitive file classes, or accept the trade-off knowingly and document it |
| Large directory rename/move | Naive implementation re-uploads every contained file | N/A — a design review catch, not a runtime symptom | Rename/move is metadata-only: update path prefixes in Metadata DB; chunk data is untouched |
| Hot partition (power user, millions of files under one user_id) | Metadata DB partition overloaded | Per-partition QPS/size monitoring | Sub-partition by path-prefix hash within the user, similar to the frontier's per-domain sub-queue split for hot domains |
| Rolling-hash CDC pathological input | Degenerate files (e.g. all-zero) produce abnormally large or small chunks | Chunk size distribution monitoring | Enforce min/max chunk size bounds regardless of rolling-hash boundary |
| Heavy-activity namespace (bulk import, large shared folder, device back after days offline) | `GET /namespaces/{id}/changes` response balloons to thousands of entries — large payload, client-side processing stall, memory pressure on constrained (mobile) clients | Response size / entry-count monitoring on the endpoint | `limit` + `has_more` cursor pagination ([Section 5](#5-apis), #5) caps every page; client loops until caught up instead of receiving one unbounded dump |

---

## 18. Scalability & Sharding

*(See the dedicated Sharding Strategies guide for the general partitioning toolkit; this section applies it to Dropbox's specific hot spots.)*

### Metadata DB Sharding

```
Primary shard key: hash(user_id)
  • Collocates a user's file tree → cheap "list files" queries
  • Even distribution for the common case (most users: modest file counts)

Hot-user mitigation (power users, shared team namespaces):
  • Sub-partition by (user_id, path_prefix_hash) once a user's
    partition exceeds a size/QPS threshold
  • Shared namespaces get their OWN partition keyed by namespace_id,
    not folded into any single member's user_id partition
```

### Block Storage (S3) Sharding

```
Shard key: chunk_hash prefix (first N hex chars)
  • Naturally uniform distribution (hash output is ~uniform)
  • No hot-spot risk from popular files, since the KEY is content-
    derived, not filename-derived
  • Scales by adding more storage nodes / object-store partitions;
    fully stateless from the routing perspective
```

### Notification Fleet Sharding

```
Shard key: hash(namespace_id)
  • All subscribers to one namespace's changes are handled by
    connection servers subscribed to the same Kafka partition
  • A single hot namespace (huge shared folder) can still be
    split further using the fan-out-on-read escape hatch
    from Section 15
```

### Horizontal Scaling Summary

| Component | Scaling Strategy |
|---|---|
| Chunk Brokers | Stateless; add nodes behind LB |
| Block storage | Managed object store; scales automatically; shard by chunk hash prefix |
| Metadata DB | Shard by user_id, sub-shard hot users by path prefix; shared namespaces get dedicated shards |
| Metadata cache | Redis cluster, consistent hashing |
| Notification fleet | Shard by namespace_id via Kafka partitioning |
| Message queue | Kafka; partition count scales with namespace cardinality |

---

## 19. Deep Dive: Control Plane vs Data Plane Split

This is the idea that, if you only communicate one thing in a 45-minute interview, should be this one — it's the load-bearing architectural decision everything else in this guide hangs off of.

```
CONTROL PLANE (Metadata Service + Metadata DB)
  • Small records (KBs, not MBs)
  • High consistency requirement (CAS on version, no silent forks)
  • Moderate QPS (hundreds to low thousands/sec at this scale)
  • CP-leaning, tunable per-operation (write vs read, Section 16)

DATA PLANE (Chunk Broker + Block Storage (S3))
  • Large payloads (MBs per chunk)
  • Content-addressed, immutable → conflict-free by construction
  • High bandwidth requirement (GB/s)
  • AP-leaning; durability handled orthogonally via replication
```

**Why splitting them matters:** if you tried to run both through the same consistency model, you'd either (a) pay CP-level coordination overhead on every multi-GB chunk write — crushing throughput — or (b) accept AP-level looseness on the metadata version chain — risking silent version forks and real data-loss-adjacent bugs. Splitting lets each plane use the consistency model its actual access pattern needs.

**The corollary that Staff candidates surface unprompted:** because chunks are immutable and content-addressed, the data plane is *structurally* free of the conflict problem — two devices can never "conflict" on a chunk, only on which ordered list of chunks constitutes the *current version* of a file. That's a metadata-plane question by construction. This is why Section 14's conflict resolution lives entirely in the Metadata Service and never touches Block Storage (S3) at all.

---

## 20. Senior vs Staff Answer Differentiators

### Senior-level expectations

- Correctly identify the major components (client, metadata service, block storage, sync/notification)
- Explain chunking at a basic level (split file into blocks)
- Mention deduplication by content hash
- Handle the obvious failure case (client crash mid-upload → resume)
- Provide basic capacity math
- Know that conflicts need "some" resolution strategy

### Staff-level expectations (additional)

| Topic | What Staff-level adds |
|---|---|
| Chunking | Content-defined chunking via rolling hash, and WHY (insertion/deletion resilience) vs fixed-size |
| Dedup | The explicit tension with per-user encryption; convergent encryption trade-off and its confirmation-attack weakness |
| Metadata writes | CAS-on-version as the narrow coordination point; explicit CP posture on the write path only |
| Metadata reads | Explicit AP-via-cache posture on the read path — same store, different posture per operation |
| Conflict resolution | Conflicted-copy forking as a deliberate choice over auto-merge, with reasoning about why CRDTs/OT don't fit arbitrary binary files |
| Sharing/fan-out | Fan-out-on-write vs fan-out-on-read hybrid for large shared namespaces (explicit cross-reference to the celebrity/fan-out problem class) |
| CDN usage | Recognizing CDN only pays off for "hot" (shared/popular) chunks, not every private download; and that content-addressed immutability eliminates the cache-invalidation problem a generic CDN design has to solve ([Section 9](#9-component-3-block-storage-service)) |
| Rename/move | Recognizing it as a metadata-only operation — a common naive-design trap |
| Control/data plane split | Explicitly separating the two planes and justifying it by differing consistency needs, not just "for organization" |
| CAP theorem | Per-OPERATION (not just per-component) CAP posture — same metadata store, different posture for reads vs writes |
| Failure modes | Proactively enumerate thundering herd on reconnect, hot-partition power users, orphaned chunk GC, degenerate CDC inputs |

### The single most important Staff differentiator

**Recognizing that "metadata" and "bytes" are not one problem but two, with genuinely different consistency requirements — and architecting the system so each gets the consistency model it actually needs, rather than applying one uniform policy everywhere.** Every other Staff-level nuance in this guide (CAS writes, cached reads, conflicted copies, content-addressed dedup) is a downstream consequence of getting this split right at the start.

---

## 21. Interview Time Allocation

For a 45-minute system design session:

| Phase | Time | Focus |
|---|---|---|
| Requirements & scope | 5 min | Confirm functional + non-functional; call out durability ≠ availability early |
| Capacity estimation | 5 min | Storage, chunk throughput, metadata QPS |
| High-level architecture | 5 min | Draw control plane / data plane split; name Kafka as the bridge |
| Chunking & dedup (deep dive) | 8 min | CDC vs fixed-size; global dedup vs encryption tension |
| Metadata service (deep dive) | 8 min | Schema, CAS-on-version, per-operation CAP posture |
| Sync/notification service | 6 min | Pub-sub fan-out, ping-not-payload, reconnect thundering herd |
| Conflict resolution | 5 min | Conflicted-copy strategy, why not CRDTs here |
| Failure modes & sharding | 3 min | Hit 2–3 proactively; mention sharding strategy briefly |

**What to cut if short on time:** sharing/permissions detail, deep sharding mechanics. **Never cut:** chunking strategy, the control/data plane split, and conflict resolution — these three are the load-bearing Staff-level signals for this specific problem.

---

## 22. Quick-Reference Cheatsheet

```
KEY ALGORITHMS
──────────────
Chunking:        Content-Defined Chunking (rolling hash / Rabin fingerprint)
                 avg 4MB chunks, resilient to byte insertion/deletion
Chunk identity:  SHA-256(chunk_bytes) — content-addressed storage key
Dedup:           Global, keyed by chunk hash; tension with per-user encryption
Version control: CAS on current_version at commit time (optimistic concurrency)
Conflict:        Conflicted-copy fork, NOT auto-merge (arbitrary binary content)
Fan-out:         Fan-out-on-write (default) / fan-out-on-read (huge shared folders)
CDN:             Presigned CDN edge URL for "hot" chunks only (shared/high ref_count);
                 private/cold chunks go direct to Block Storage (S3). No invalidation
                 needed — chunks are immutable, so cache residency, not correctness

KEY NUMBERS (illustrative)
───────────────────────────
500M users, 100M DAU
2.5 EB logical storage → ~1.9 EB physical after global dedup
~1,000 chunks/sec average ingest, ~3,000/sec peak
~4 GB/s average ingest bandwidth, ~12 GB/s peak
~350 metadata writes/sec average
100 TB+ metadata store → sharding mandatory

KEY SYSTEMS
───────────
Control plane:   Metadata Service + sharded Metadata DB (Cassandra/DynamoDB-style)
                 + Redis cache for hot reads
Data plane:      Chunk Broker + content-addressed Object Store (S3-style)
Bridge:          Kafka (durable change events) → Notification Service (pub-sub)
Client:          Watcher + Chunker + Indexer + local SQLite state

CAP / PACELC DECISIONS
───────────────────────
CP:  Metadata write path (CAS on version), Sharing/ACL store
AP:  Metadata read path (cached), Block storage, Notification service
Same store, different posture per OPERATION: metadata DB writes vs reads

STAFF-LEVEL SIGNALS TO HIT
───────────────────────────
✓ Control plane / data plane split, explicitly justified
✓ Content-defined chunking and WHY over fixed-size
✓ Dedup-vs-encryption tension (convergent encryption trade-off named)
✓ CAS-on-version as the narrow coordination point (not a heavy lock)
✓ Conflicted-copy resolution, with reasoning against CRDTs/OT here
✓ Fan-out-on-write/read hybrid for large shared namespaces
✓ Rename/move as metadata-only (no re-upload)
✓ Per-OPERATION CAP posture, not just per-component
✓ Reconnect thundering herd + hot-partition power users called out proactively
```

---

*Guide compiled for Senior & Staff-level system design interview preparation.*
*Topics: Dropbox, Distributed File Sync, Content-Defined Chunking, Deduplication, Conflict Resolution, CAP Theorem.*
*Cross-references: Kafka guide (pub-sub fan-out mechanics), Twitter/Fan-out guide (fan-out-on-write vs read), Sharding Strategies guide (partitioning toolkit), CAP/PACELC guide (general framework), Distributed Key-Value Store guide (quorum/replication for block storage).*
