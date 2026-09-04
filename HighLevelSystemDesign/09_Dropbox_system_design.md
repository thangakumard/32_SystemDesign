# Dropbox — System Design Interview Guide
### Senior & Staff Engineer Level

---

## Table of Contents

1. [Problem Statement & Scope](#1-problem-statement--scope)
2. [Requirements](#2-requirements)
3. [Capacity Estimation](#3-capacity-estimation)
4. [High-Level Architecture](#4-high-level-architecture)
5. [Component 1: Client (Watcher, Chunker, Indexer)](#5-component-1-client-watcher-chunker-indexer)
6. [Component 2: Chunking Strategy](#6-component-2-chunking-strategy)
7. [Component 3: Block Storage Service](#7-component-3-block-storage-service)
8. [Component 4: Metadata Service & Metadata DB](#8-component-4-metadata-service--metadata-db)
9. [Component 5: Synchronization & Notification Service](#9-component-5-synchronization--notification-service)
10. [Component 6: Upload & Download Flow](#10-component-6-upload--download-flow)
11. [Component 7: Deduplication (and the Encryption Tension)](#11-component-7-deduplication-and-the-encryption-tension)
12. [Component 8: Conflict Resolution & Versioning](#12-component-8-conflict-resolution--versioning)
13. [Component 9: Sharing & Permissions](#13-component-9-sharing--permissions)
14. [CAP / PACELC Positioning](#14-cap--pacelc-positioning)
15. [Failure Modes & Mitigations](#15-failure-modes--mitigations)
16. [Scalability & Sharding](#16-scalability--sharding)
17. [Deep Dive: Control Plane vs Data Plane Split](#17-deep-dive-control-plane-vs-data-plane-split)
18. [Senior vs Staff Answer Differentiators](#18-senior-vs-staff-answer-differentiators)
19. [Interview Time Allocation](#19-interview-time-allocation)
20. [Quick-Reference Cheatsheet](#20-quick-reference-cheatsheet)

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

## 4. High-Level Architecture

```
                         ┌──────────────────────────┐
                         │   Client                  │
                         │   Watcher · Chunker        │
                         │   Indexer · Local SQLite   │
                         └────────────┬───────────────┘
                                      │  HTTPS
                         ┌────────────▼───────────────┐
                         │   API Gateway / LB          │
                         └───────┬──────────┬──────────┘
                                 │          │
                 ┌───────────────▼──┐   ┌───▼─────────────────────┐
                 │  Metadata Service │   │  Block Server            │
                 │  (file tree,      │   │  dedup check · compress  │
                 │   versions, ACLs) │   │  encrypt · chunk commit  │
                 └───┬───────────┬───┘   └────────────┬─────────────┘
                     │           │                     │
         ┌───────────▼──┐   ┌────▼─────┐   ┌───────────▼────────────┐
         │ Metadata DB   │   │ Metadata │   │  Block Storage          │
         │ (sharded,     │   │ Cache    │   │  (Object Store — S3)    │
         │  e.g. by      │   │ (Redis)  │   │  content-addressed      │
         │  user_id/     │   └──────────┘   │  by chunk hash          │
         │  file_id)     │                  └──────────────────────────┘
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
```

**The one-line architectural summary:** metadata flows through a **strongly-consistent, sharded control plane**; bytes flow through a **content-addressed, highly-available data plane**; and a **durable pub-sub layer** bridges the two to tell other devices "something changed, go pull the new metadata." This split is elaborated in Section 17.

---

## 5. Component 1: Client (Watcher, Chunker, Indexer)

The client is not a thin uploader — it's a meaningful piece of the distributed system, responsible for detecting change, minimizing bytes sent, and surviving being offline.

### Sub-components

```
Watcher:
  • OS-level file system event hooks (inotify on Linux,
    FSEvents on macOS, ReadDirectoryChangesW on Windows)
  • Debounces rapid-fire events (e.g. an editor doing many
    small writes while saving) into a single "file changed" signal

Chunker:
  • Splits the changed file into content blocks (see Section 6)
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

## 6. Component 2: Chunking Strategy

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
this is what makes global deduplication possible (Section 11).
```

---

## 7. Component 3: Block Storage Service

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

### Block Server — the gatekeeper in front of storage

```
Client → Block Server:
  1. Client sends chunk_hash (not the bytes yet)
  2. Block Server checks: does this hash already exist in
     block storage? (dedup check — see Section 11)
  3a. Exists → respond "skip upload", increment ref_count
  3b. Doesn't exist → respond "upload", client streams bytes
  4. Block Server compresses + encrypts + writes to object store
  5. Block Server commits chunk record, returns success
  6. Client reports chunk as "synced" to Metadata Service
```

This two-phase "ask before you send" protocol is what makes delta sync + global dedup actually save bandwidth — the client never uploads bytes the server already has.

---

## 8. Component 4: Metadata Service & Metadata DB

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

Partitioning by `user_id` collocates a user's whole file tree — cheap "list my files" queries — at the cost of potential hot partitions for power users with millions of files (mitigated with sub-partitioning by path prefix, see Section 16).

### The Write Path Needs a Single-Writer Guarantee Per File

Two devices editing the same file concurrently must not both win — the metadata write path needs **conditional writes** (compare-and-swap on `current_version`):

```
UPDATE file_metadata
SET current_version = new_version, chunk_hashes = ...
WHERE user_id = ? AND file_id = ? AND current_version = expected_version
IF current_version = expected_version   -- CAS / optimistic lock

If the CAS fails → another device already advanced the version →
this is a genuine conflict → route to conflict resolution (Section 12),
do NOT silently overwrite.
```

This CAS-on-version-number is the metadata-layer equivalent of the crawler guide's "distributed lock via Redis SET NX" — a narrow, cheap coordination point rather than a heavyweight distributed lock across the whole write path.

---

## 9. Component 5: Synchronization & Notification Service

### The Problem

When device A writes a new version, devices B and C (if online) need to find out — ideally within seconds, without every device polling the server every few seconds (which doesn't scale to 100M DAU).

### Design: Pub-Sub Fan-Out, Not Payload Push

```
Block Server / Metadata Service, on successful commit:
  → publish {namespace_id, file_id, new_version} to Kafka topic

Notification Service (Kafka consumer):
  → looks up which devices are currently subscribed to namespace_id
  → pushes a lightweight "namespace changed" ping to each
    (via long-lived WebSocket, or long-poll for constrained clients)

Client, on receiving the ping:
  → does NOT receive the file bytes in the push
  → calls Metadata Service: "what changed since my last known version?"
  → pulls only the metadata delta, then requests only the chunks
    it doesn't already have locally
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

---

## 10. Component 6: Upload & Download Flow

### Upload (small-to-medium file)

```
1. Watcher detects change → Chunker splits into content-defined chunks
2. Client asks Metadata Service to start a new version, gets version token
3. For each chunk: ask Block Server "do you have hash H?" (dedup check)
4. Upload only missing chunks (parallelized, several chunks concurrently)
5. Once all chunks acknowledged → client commits new version to
   Metadata Service with the ordered chunk hash list + CAS on version
6. Metadata Service publishes change event → Notification Service fans out
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
4. Client downloads only missing chunks directly from Block Storage
   (pre-signed URLs, direct-to-object-store — Block Server is not
   a bandwidth bottleneck in the read path)
5. Client reassembles file from chunks in order, writes to disk
6. Client updates local DB with new version + chunk list
```

**Why pre-signed URLs matter:** routing every chunk download through the Block Server as a proxy makes it a bandwidth chokepoint. Pre-signed, time-limited URLs let clients pull directly from the object store's edge/CDN layer, and the Block Server's job stays limited to authorization + metadata bookkeeping.

---

## 11. Component 7: Deduplication (and the Encryption Tension)

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

## 12. Component 8: Conflict Resolution & Versioning

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

## 13. Component 9: Sharing & Permissions

### Shared Folder as a Namespace

```
namespace_id ≠ user_id — a shared folder is its own namespace with
its own member list and ACL, referenced by symlink-like entries in
each member's personal file tree.

folder_members: { namespace_id, user_id, role (owner/editor/viewer),
                   joined_at }
```

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
    folder_members for that namespace
  • Block Storage reads use pre-signed URLs scoped and
    time-limited per authorized request — never a durable,
    shareable direct link to raw storage
```

---

## 14. CAP / PACELC Positioning

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

## 15. Failure Modes & Mitigations

| Failure | Symptom | Detection | Mitigation |
|---|---|---|---|
| Client crash mid-upload | Partial chunk set uploaded, version never committed | Client's local DB shows "pending" chunks on restart | Chunking makes upload naturally resumable; re-run dedup-check step, only missing chunks re-sent; version never partially visible to other devices (commit is atomic) |
| Concurrent edits from two devices | CAS failure on version commit | Metadata Service returns version-mismatch error | Conflicted-copy fork (Section 12); never silently overwrite |
| Metadata DB partition | Risk of split-brain version forks | Write quorum failure / leader election events | CP posture on write path: reject writes on the minority side rather than risk divergent "current" versions |
| Notification service outage | Clients don't get real-time pings | Kafka consumer lag on notification topic; client heartbeat misses | Clients fall back to periodic full metadata poll (e.g. every few minutes) as backstop; reconnect triggers immediate metadata pull |
| Reconnect thundering herd | Metadata DB read spike after outage recovery | Sudden QPS spike correlated with notification-service recovery | Jittered client reconnect backoff; Redis cache absorbs the read spike |
| Chunk storage node failure | Risk of losing a chunk that many files reference | Storage replication health checks | Replication factor 3 (or erasure coding); a single lost replica is a non-event |
| Orphaned chunks after delete | Wasted storage cost | Periodic ref-count audit job | Reference counting; garbage collect only when ref_count hits 0 across ALL files/versions |
| Convergent-encryption confirmation attack | Attacker infers a user has a known file | N/A (architectural property, not an incident) | Explicitly scope cross-user dedup to non-sensitive file classes, or accept the trade-off knowingly and document it |
| Large directory rename/move | Naive implementation re-uploads every contained file | N/A — a design review catch, not a runtime symptom | Rename/move is metadata-only: update path prefixes in Metadata DB; chunk data is untouched |
| Hot partition (power user, millions of files under one user_id) | Metadata DB partition overloaded | Per-partition QPS/size monitoring | Sub-partition by path-prefix hash within the user, similar to the frontier's per-domain sub-queue split for hot domains |
| Rolling-hash CDC pathological input | Degenerate files (e.g. all-zero) produce abnormally large or small chunks | Chunk size distribution monitoring | Enforce min/max chunk size bounds regardless of rolling-hash boundary |

---

## 16. Scalability & Sharding

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

### Block Storage Sharding

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
    from Section 13
```

### Horizontal Scaling Summary

| Component | Scaling Strategy |
|---|---|
| Block servers | Stateless; add nodes behind LB |
| Block storage | Managed object store; scales automatically; shard by chunk hash prefix |
| Metadata DB | Shard by user_id, sub-shard hot users by path prefix; shared namespaces get dedicated shards |
| Metadata cache | Redis cluster, consistent hashing |
| Notification fleet | Shard by namespace_id via Kafka partitioning |
| Message queue | Kafka; partition count scales with namespace cardinality |

---

## 17. Deep Dive: Control Plane vs Data Plane Split

This is the idea that, if you only communicate one thing in a 45-minute interview, should be this one — it's the load-bearing architectural decision everything else in this guide hangs off of.

```
CONTROL PLANE (Metadata Service + Metadata DB)
  • Small records (KBs, not MBs)
  • High consistency requirement (CAS on version, no silent forks)
  • Moderate QPS (hundreds to low thousands/sec at this scale)
  • CP-leaning, tunable per-operation (write vs read, Section 14)

DATA PLANE (Block Server + Block Storage)
  • Large payloads (MBs per chunk)
  • Content-addressed, immutable → conflict-free by construction
  • High bandwidth requirement (GB/s)
  • AP-leaning; durability handled orthogonally via replication
```

**Why splitting them matters:** if you tried to run both through the same consistency model, you'd either (a) pay CP-level coordination overhead on every multi-GB chunk write — crushing throughput — or (b) accept AP-level looseness on the metadata version chain — risking silent version forks and real data-loss-adjacent bugs. Splitting lets each plane use the consistency model its actual access pattern needs.

**The corollary that Staff candidates surface unprompted:** because chunks are immutable and content-addressed, the data plane is *structurally* free of the conflict problem — two devices can never "conflict" on a chunk, only on which ordered list of chunks constitutes the *current version* of a file. That's a metadata-plane question by construction. This is why Section 12's conflict resolution lives entirely in the Metadata Service and never touches Block Storage at all.

---

## 18. Senior vs Staff Answer Differentiators

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
| Rename/move | Recognizing it as a metadata-only operation — a common naive-design trap |
| Control/data plane split | Explicitly separating the two planes and justifying it by differing consistency needs, not just "for organization" |
| CAP theorem | Per-OPERATION (not just per-component) CAP posture — same metadata store, different posture for reads vs writes |
| Failure modes | Proactively enumerate thundering herd on reconnect, hot-partition power users, orphaned chunk GC, degenerate CDC inputs |

### The single most important Staff differentiator

**Recognizing that "metadata" and "bytes" are not one problem but two, with genuinely different consistency requirements — and architecting the system so each gets the consistency model it actually needs, rather than applying one uniform policy everywhere.** Every other Staff-level nuance in this guide (CAS writes, cached reads, conflicted copies, content-addressed dedup) is a downstream consequence of getting this split right at the start.

---

## 19. Interview Time Allocation

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

## 20. Quick-Reference Cheatsheet

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
Data plane:      Block Server + content-addressed Object Store (S3-style)
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
