# Dropbox — System Design

*Format follows the [hellointerview-style](https://www.hellointerview.com/learn/system-design/problem-breakdowns/dropbox) problem breakdown: requirements → entities → APIs → high-level design walked one functional requirement at a time (bad → good → great), then deep dives.*

---

## 1. Functional Requirements

**Core (in scope)**

- Users should be able to upload a file from any device.
- Users should be able to download a file from any device.
- Users should be able to share a file with other users.
- Users should be able to automatically sync files across devices.

**Out of scope**

- Real-time collaborative editing (Google Docs-style, character-level merge).
- Commenting, file previews, search/indexing of file contents.
- Billing/storage-quota management UI.

---

## 2. Non-Functional Requirements

**Core**

- **Durability** above all else — losing a user's file is never acceptable. (11 nines.)
- **Availability** — the system should be available for upload/download/sync, but brief unavailability is tolerable (unlike durability). CAP-wise, we favor **AP**: it's okay if a user in Germany uploads a file and a user in the US can't see it for a few seconds.
- **Scalability** — ~500M registered users, ~100M DAU, files up to 50 GB.
- **Low latency** — sync should propagate small changes to other online devices within a few seconds.
- **Storage efficiency** — don't store duplicate bytes; don't re-upload a whole file for a one-line edit.

**Out of scope**

- Strict/strong global consistency on every read path.
- Legal/compliance (data residency, retention law) — acknowledged but not designed in detail.

---

## 3. Core Entities

| Entity | Description |
|---|---|
| **File** | The raw bytes a user uploads. In practice broken into **Chunks** for storage/sync. |
| **FileMetadata** | Name, size, MIME type, owner, current version, folder path. The thing the Metadata Service manages. |
| **Chunk** | A content-addressed slice of a file's bytes (hash = identity); the unit of upload, download, delta-sync and dedup. |
| **User** | Account; owns files/folders, is a member of shared namespaces. |
| **Device** | One client install for a user; tracks its own last-synced cursor. |

```
User ──1:N── File ──1:N── FileVersion ──N:M── Chunk
User ──1:N── Device
File ──N:M── User   (sharing, via a Namespace/ACL)
```

---

## 4. APIs

Client-facing, REST-ish over HTTPS (mirrors the reference page's endpoint set, extended slightly for versioning/sync):

```
POST   /files/presigned-url
  → { uploadUrl, fileId }
  Returns a presigned URL the client uploads a chunk's bytes directly to
  (blob storage), plus the fileId/chunk record to commit against later.

GET    /files/{fileId}/presigned-url
  → { downloadUrl }
  Returns a presigned URL (S3, or CDN-signed — see 5.2) to download a file/chunk.

PATCH  /files/{fileId}/chunks
  Request: { chunks: [{ id, status }] }
  Client reports which chunks it has staged; also how the server flips a
  chunk's status uploading → uploaded once blob storage confirms receipt.

POST   /files/{fileId}/share
  Request: { users: [User], role }
  → shares a file/folder with other users at a given permission level.

GET    /files/changes?since={cursor}
  → ChangeEvent[]
  Polled (or pushed via WS) by a client to learn what changed since its
  last known cursor — the core of cross-device sync.
```

---

## 5. High-Level Design

### 5.1 Users should be able to upload a file from any device

**Bad — upload the whole file through our backend**

```
Client --PUT whole file bytes--> Backend --PUT--> Blob Storage (S3)
```
Every byte makes two network hops and the backend's bandwidth scales with total upload traffic — it becomes the bottleneck. A dropped connection mid-upload re-sends the *entire* file.

**Good — chunk the file, upload chunks directly to blob storage**

```
1. Client splits the file into chunks, hashes each one (SHA-256)
2. POST /files/presigned-url (per chunk) → presigned PUT url
3. Client PUTs chunk bytes STRAIGHT to Blob Storage (S3) — backend not in this hop
4. Blob storage confirms receipt (event notification) → backend flips
   that chunk's status to "uploaded" via the same mechanism as PATCH /files/{fileId}/chunks
5. Once all chunks for a version are "uploaded", client commits the
   file's new version (ordered chunk-hash list) to the Metadata Service
```
This fixes the bandwidth bottleneck (backend only ever brokers hashes + short-lived URLs) and makes resumability free: a crash mid-upload just means re-running the dedup/presign step for whatever chunks never landed.

**Great — dedup before uploading, and make chunk boundaries resilient to edits**

```
Before step 3 above: GET /files/{fileId}/presigned-url style check —
"do you already have hash H?" If a chunk's content already exists
anywhere in blob storage (same bytes, any user), skip the upload
entirely and just bump a reference count.

Use content-defined chunking (rolling hash) instead of fixed-offset
chunking, so inserting one byte near the start of a file only changes
the one or two chunks touching the edit — not every chunk after it.
```
This is what makes a 2GB-file, one-frame edit a ~4MB re-upload instead of a 2GB one, and lets the *same* bytes shared across many users (a common PDF, an OS installer) only ever be stored once.

---

### 5.2 Users should be able to download a file from any device

**Bad — proxy the download through our backend**

```
Client --GET--> Backend --GET--> Blob Storage (S3) --bytes--> Backend --bytes--> Client
```
Same problem as uploads in reverse: every downloaded byte transits the backend twice, making it the throughput bottleneck for the whole system's read traffic.

**Good — direct presigned download from blob storage**

```
1. Client: GET /files/{fileId}/presigned-url
2. Backend checks permissions, then returns a short-lived, signed
   download URL pointing directly at the object in Blob Storage (S3)
3. Client downloads the bytes directly from S3 — backend is out of the hop
```
This removes the backend as a bandwidth chokepoint — it only ever authorizes and signs URLs. The limitation: a client far from the S3 region (e.g. Asia, bucket in us-east-1) still pays cross-region latency on every download.

**Great — serve popular/shared files from a CDN in front of blob storage**

```
1. Client: GET /files/{fileId}/presigned-url
2. Backend decides URL flavor:
     - rarely-accessed / private file → presigned S3 URL (as in "Good")
     - frequently-accessed / shared file → presigned CDN-signed URL
3. CDN edge serves the file from the nearest location to the client
     - cache MISS → CDN pulls from S3 origin once, then caches it
     - cache HIT  → served straight from the edge, S3 never touched
```
The CDN (e.g. CloudFront) verifies the signed URL's cryptographic signature and short expiration window (e.g. 5 minutes) before serving — same security model as the direct-S3 presigned URL, just fronted by edge caching. CDNs cost money per byte cached/served, so this is opt-in for files that are actually re-requested from many locations (shared files, popular content) — not the default for every private, one-user download. Cache invalidation is simple here because files/chunks are content-addressed and immutable: editing a file produces new chunks with new hashes, never a mutated object at the same cached URL, so a short TTL plus normal cache-control headers is enough — there's no "stale CDN copy" correctness problem to solve.

---

### 5.3 Users should be able to share a file with other users

```
POST /files/{fileId}/share
Request: { users: [User], role: owner|editor|viewer }
```

```
1. Sharing a file/folder writes a row associating the file/folder's
   Namespace with the target user and a role (owner/editor/viewer)
2. The shared item appears in the recipient's own file listing via
   a reference — no separate "shared with me" query path needed
3. Every subsequent read/write against that item is checked against
   this membership+role record at the backend — never trusted to
   the client to self-restrict
4. Downloads still go through the presigned-URL flow in 5.2 — we
   never hand out a durable, un-expiring link to the raw object
```
At scale, a share to a very large group (e.g. 10,000-person team folder) changes how change-notifications fan out (see sync deep dive) — fan-out-on-write for small shares, fan-out-on-read for very large ones — but the sharing *data model* above doesn't change.

---

### 5.4) Users can automatically sync files across devices

Sync is two independent flows glued together by `GET /files/changes?since={cursor}`.

#### Local → Remote (this device pushes a change it just made)

```
1. Watcher detects a local file change (OS-level FS event hooks)
2. Chunker splits + hashes the changed content (5.1)
3. Client uploads any missing chunks (5.1's upload flow)
4. Client commits the new file version to the Metadata Service
   (compare-and-swap on the file's current version — if another
   device already advanced it, this is a genuine conflict, not a
   silent overwrite)
5. Backend records a ChangeEvent so other devices can discover it
```

#### Remote → Local (other devices find out and catch up)

```
1. A device subscribed to this file/folder gets pushed a lightweight
   "something changed, go look" notification (WebSocket/long-poll) —
   the payload is NOT the file bytes, just a signal
2. Device calls GET /files/changes?since={my_last_cursor}
   → ChangeEvent[] { fileId, newVersion, ... }, new cursor
3. For each changed file, device fetches the new version's chunk list,
   diffs it against chunks it already has locally, and downloads only
   what's missing (5.2's download flow)
4. Device reassembles the file, writes it to disk, advances its cursor
```

**Why the cursor, not just the push, is the correctness guarantee:** the push notification is best-effort — if a device is offline when it fires, nothing re-sends it. On reconnect, the device simply replays `GET /files/changes?since={cursor}` from wherever it last left off, so a missed/delayed notification never causes a missed sync — only added latency until the device happens to reconnect or poll.

---

### Putting it together

```
                         ┌─────────────┐
                         │   Backend    │ (metadata, auth, presigned URLs,
                         │  (stateless) │  share/ACL, change feed)
                         └──────┬───────┘
             issues presigned URLs │ change events
          ┌─────────────────────┼─────────────────────┐
          ▼                     ▼                      ▼
   Blob Storage (S3)      Metadata DB            Notification channel
   (chunks, content-      (files, versions,      (WS/long-poll — "go
    addressed)             ACLs)                  check for changes")
          ▲
          │ CDN in front, for
          │ hot/shared files only (5.2)
   ┌──────┴──────┐
   │     CDN      │
   └─────────────┘
```

---

## 6. Potential Deep Dives

### 6.1 How can you support large files?

Large files are "just many chunks" — chunking already buys resumability and parallelism for free:

- Split into content-defined chunks (e.g. ~4–8 MB average) rather than one giant upload.
- Upload chunks **in parallel** (several concurrent presigned PUTs).
- If the client crashes mid-upload, on restart it re-checks which chunks already landed (the dedup/exists check) and only sends what's missing — no separate "multipart upload" protocol needed beyond what chunking already gives you.

### 6.2 How should we ensure the `chunks` field stays in sync with the chunks actually uploaded?

The naive design lets the *client* say "I'm done uploading" — but a client can crash right after its `PUT` succeeds and never report back, leaving the metadata permanently unaware the bytes exist.

```
Fix: make the confirmation infra-pushed, not client-pushed.

1. Blob storage fires an event notification when an object is created
   (e.g. S3 Event Notification → SQS/SNS)
2. The backend consumes that queue — NOT a client callback — and is
   what actually flips a chunk's status from "uploading" → "uploaded"
3. PATCH /files/{fileId}/chunks is used for the client to report
   staged/expected chunks, but the status transition to "uploaded"
   is only ever trusted from the storage-layer notification
4. A periodic reconciliation sweep catches stragglers: any chunk
   still "uploading" past its presigned URL's expiry gets a HEAD
   check against blob storage — object present → commit it (the
   notification was just slow/lost); object absent → delete the
   stale record so the hash can be retried
```
This makes "bytes landed in S3 but the record never got created" a structurally unreachable failure mode rather than something that needs a client-honesty assumption.

### 6.3 How can we make uploads, downloads, and syncing as fast as possible?

- **Uploads:** parallel chunk upload (6.1); skip re-uploading bytes the server already has (dedup, 5.1); content-defined chunking so small edits only re-upload the chunks touching the edit, not the whole file.
- **Downloads:** direct-to-blob-storage presigned URLs, never proxied through the backend (5.2); CDN edge caching for hot/shared files (5.2) so repeat/nearby requests never leave the edge.
- **Syncing:** push a tiny "go check" ping instead of the payload (keeps the notification layer thin and fast to fan out); clients diff against their own local chunk cache before downloading anything, so a device that already has most of a file's chunks (e.g. from a prior version) only pulls the delta.

### 6.4 How can you ensure file security?

- **Encryption in transit** — all client ↔ backend and client ↔ blob-storage/CDN traffic over TLS; presigned URLs are themselves short-lived, signed, and scoped to one object/operation (typically minutes), so a leaked URL has a small blast radius and expires on its own.
- **Encryption at rest** — chunks are encrypted before/at storage (server-side encryption on the bucket at minimum; client-side/convergent encryption if cross-user dedup must survive per-user keys — convergent encryption trades in a known "confirmation of file" side channel, worth calling out explicitly rather than glossing over).
- **Access control** — every metadata read/write and every presigned-URL issuance is checked against the file/folder's ACL (5.3) at the backend; the client is never trusted to self-restrict, and there's never a durable, un-expiring direct link to raw storage handed to anyone.

---

*Companion to [`02_Dropbox_system_design.md`](./02_Dropbox_system_design.md) (the detailed Staff-level guide) — this version mirrors the condensed hellointerview-style walkthrough structure from [`02_Dropbox_design_approach.txt`](./02_Dropbox_design_approach.txt).*
