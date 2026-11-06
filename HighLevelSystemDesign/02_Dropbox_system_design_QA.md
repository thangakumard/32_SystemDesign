# Dropbox System Design — Q&A Log

Companion Q&A notes for [`02_Dropbox_system_design.md`](./02_Dropbox_system_design.md), captured from a working session covering schema review, naming/API cleanup, and interview-style deep-dive questions.

---

## 1. Is the Chunk table a child of the Version table?

No — it's a many-to-many relationship, not parent-child.

```
Namespace ──1:N── File ──1:N── Version (ordered, immutable, append-only)
Version ──N:M── Chunk (ordered hash list)  ──1:1── Block (physical bytes)
```

- `Namespace → File → Version` is a true 1:N parent/child chain.
- `Version → Chunk` is **N:M**: a Version references an ordered list of chunk hashes (`chunk_hash_list[]`), and any given Chunk can be referenced by many Versions — that's exactly what enables dedup (a chunk unchanged between two versions, or shared across different users' files, isn't duplicated).

Chunks also carry their own independent lifecycle (`ref_count`, `status: uploading|uploaded`) rather than being owned/deleted alongside a single Version. Deleting a Version just decrements `ref_count` on the chunks it referenced, not deleting the chunks themselves.

---

## 2. Why should Block be a separate table instead of keeping `storage_location` in the Chunk table? What does `storage_location` actually store?

**Why split them (even though it's 1:1 today):**

1. **Different write pattern / hotness.** `Chunk` (hash, size, ref_count, status) is hit on every upload via `CHUNK_EXISTS` and has `ref_count` constantly incremented/decremented. `Block` (storage_location) is written once and rarely touched again. Bundling a rarely-changed physical pointer into a row under constant CAS contention adds unnecessary write pressure to the hot path.
2. **Logical identity vs. physical placement are different concerns.** `Chunk` answers "does this content exist and how many things reference it" (control-plane-ish, owned by the Chunk Broker). `Block` answers "where do the bytes physically live" (storage-infra concern). Keeping them separate means a storage migration (cold tiering, region move, backend re-shard) doesn't touch the dedup/ref_count row at all.
3. **Room to go 1:N later.** Multi-region replication or hot/cold tiering could mean one chunk's bytes live in more than one physical location — modeled as "add another Block row," not a schema rewrite.

**What `storage_location` holds:** the physical address needed to fetch bytes — e.g. `s3://blocks/{hash[0:2]}/{hash[2:4]}/{full_hash}` (prefix-sharded key), which region/storage class, and/or a replication/erasure-coding group id. `Chunk` = "does this exist and who's using it"; `Block` = "go here to get the actual bytes."

---

## 3. Recommend a different name for the "Folder Member" table

**Recommended and applied: `NamespaceMember`.**

The table's actual key is `namespace_id` (not a folder id), and the doc is explicit that "shared folder" is just one instance of a `Namespace`. Calling it `Folder Member` introduced a second vocabulary word for a concept the rest of the schema calls `Namespace`. (Alternative considered: `NamespaceACL` — rejected because the table carries `joined_at`, which is membership-flavored, not pure-ACL-flavored.)

*Applied across the doc: entity table, relationship diagram, `SHARE_NAMESPACE`/`RESOLVE_PERMISSION` API rows, and the §15 sharing narrative (`folder_members` → `namespace_members`).*

---

## 4. What is `connection_id` in the Sync Subscription (now DeviceConnection) table?

It identifies the specific live socket a device is holding open on the notification fleet — the handle the Notification Service needs to actually push a ping to that device.

- `device_id` identifies *who* is interested; `connection_id` identifies *which currently-open socket*, on *which connection-server node*, that interest is bound to right now.
- A device can reconnect (network flap, app restart, server node restart) and get a new socket each time — still the same device, different `connection_id`.
- When Metadata Service publishes `{namespace_id, file_id, new_version}` to Kafka, the Notification Service looks up subscription rows for that namespace and uses each row's `connection_id` to know exactly which open socket to push the "namespace changed" ping down.
- This is also why the row is ephemeral ("only while online") — unlike `NamespaceMember`, it's torn down/replaced on every disconnect-reconnect cycle.

---

## 5. Recommend a different name for the "Sync Subscription" table

**Recommended and applied: `DeviceConnection`.**

Considered `NamespaceSubscription` (for naming-family consistency with `NamespaceMember`), but chose `DeviceConnection` instead because it foregrounds the table's defining property — ephemerality and socket-routing, torn down on every disconnect — which matters more here than naming-family consistency, given how different its write/delete pattern is from everything else in the schema.

---

## 6. Add a serial number column to the Client-Facing API table, keeping functional-flow order

Added `S.No` (1–12). Existing row order already followed the functional flow and was kept as-is:
1. Upload path: `CHUNK_EXISTS` → `GET_UPLOAD_URL` → `BEGIN_VERSION` → `COMMIT_VERSION`
2. Sync/download path: `GET_METADATA_DELTA` → `GET_FILE_VERSION` → `DOWNLOAD_CHUNK`
3. File lifecycle: `DELETE_FILE` → `LIST_VERSIONS` → `RESTORE_VERSION`
4. Sharing: `SHARE_NAMESPACE`
5. Notifications: `SUBSCRIBE`

---

## 7. To support a 50GB file, how many chunks get created, and how many times is `GET_UPLOAD_URL` called?

**Chunk count:** `50 GB / 4 MB ≈ 51,200 MB / 4 MB = 12,800 chunks` (average — CDC produces variable-size chunks by design, so the real count will land around 12,800, not exactly on it).

**`GET_UPLOAD_URL` calls:**
- Worst case (every chunk is novel content): **~12,800 calls**, one per chunk.
- Realistic case: fewer — any chunk that matches existing content (same file's prior version, another file, another user, via global dedup) reports `CHUNK_EXISTS: true` and skips `GET_UPLOAD_URL` entirely.
- No separate multipart-upload API is needed — a 50GB file uses the exact same per-chunk `CHUNK_EXISTS → GET_UPLOAD_URL → PUT` loop as any other file, just repeated ~12,800 times; this is also what gives it resumability for free.

---

## 8. In which API does file metadata actually get written?

**`COMMIT_VERSION`** (now `CommitFileVersion`).

- `BeginFileVersion` writes nothing durable — it just reserves a CAS baseline (`expected_current_version`).
- `CommitFileVersion(file_id, expected_version, new_version, chunk_hash_list[], size_bytes, device_id)` is the actual write: it creates the new `Version` row and advances `File.current_version` via CAS. It can return `VERSION_CONFLICT` (lost the race) or `CHUNKS_PENDING` (a referenced chunk's S3 confirmation hasn't landed yet).

Order: upload all chunk bytes first (repeated `CHUNK_EXISTS → GET_UPLOAD_URL → PUT`), **then** call `CommitFileVersion` once with the complete `chunk_hash_list[]`. Metadata is written once per version, not once per chunk.

---

## 9. Is there an alternate name for `BeginVersion`/`CommitVersion`?

Flagged a naming inconsistency: `BEGIN_VERSION`/`COMMIT_VERSION` read like opening a lock or starting a transaction — the opposite of optimistic locking's "no lock held" model (per `CoreConcepts/OptimisticLocking.md`), and inconsistent with `01_DesignKeyValueStore.md`'s `PUT_IF_VERSION_MATCHES` naming for the identical CAS pattern.

Suggested at the time: `GET_VERSION_BASELINE` (read-only baseline fetch) and `COMMIT_VERSION_IF_MATCH` (explicit CAS condition in the name). The user instead later chose **`BeginFileVersion` / `CommitFileVersion`** (see §15 below) — which also happened to fix a different inconsistency: `GetFileVersion` already used the `FileVersion` naming, so `BeginVersion`/`CommitVersion` were the odd ones out.

---

## 10. Add the HTTP method of each API to the Client-Facing API table

| Method | Assigned to |
|---|---|
| `GET` | `CHUNK_EXISTS`, `GET_METADATA_DELTA`, `GET_FILE_VERSION`, `DOWNLOAD_CHUNK`, `LIST_VERSIONS` — pure reads, no side effects |
| `POST` | `GET_UPLOAD_URL` (writes a chunk record), `BEGIN_VERSION` (reserves a CAS token), `SHARE_NAMESPACE` (creates a `NamespaceMember` row), `SUBSCRIBE` (opens a connection) — non-idempotent, server-generated-identifier actions |
| `PUT` | `COMMIT_VERSION`, `RESTORE_VERSION` — CAS-conditional updates to an existing, known resource (`RESTORE_VERSION` is explicitly "a new `COMMIT_VERSION`") |
| `DELETE` | `DELETE_FILE` — soft-delete |

Caveat noted: this is an RPC-style API (`CHUNK_EXISTS`, `COMMIT_VERSION`, etc.), not pure resource-oriented REST, so a team preferring RPC conventions might reasonably make every endpoint `POST` instead.

---

## 11. Merge the Operation and HTTP Method columns into one, e.g. `GET /ChunkExists`

Applied — `SNAKE_CASE` operation names converted to `PascalCase` path segments, merged with their method:

| S.No | Endpoint |
|---|---|
| 1 | `GET /ChunkExists` |
| 2 | `POST /GetUploadUrl` |
| 3 | `POST /BeginVersion` *(later renamed, see §15)* |
| 4 | `PUT /CommitVersion` *(later renamed, see §15)* |
| 5 | `GET /GetMetadataDelta` |
| 6 | `GET /GetFileVersion` |
| 7 | `GET /DownloadChunk` |
| 8 | `DELETE /DeleteFile` |
| 9 | `GET /ListVersions` |
| 10 | `PUT /RestoreVersion` |
| 11 | `POST /ShareNamespace` |
| 12 | `POST /Subscribe` |

---

## 12. Why is `BeginVersion` a `POST` and `CommitVersion` a `PUT`?

**`BeginVersion` → `POST`:** generates a brand-new `version_token` on every call — two identical calls return two different tokens. Non-idempotent, server-generated-identifier semantics is what `POST` is for; there's no stable resource URL for "the thing being begun" yet.

**`CommitVersion` → `PUT`:** targets a resource already identified by `file_id` and conditionally sets its state to a specific target value, guarded by a CAS check (`expected_version`) — the textbook shape of `PUT`. It's idempotent in the sense that matters: repeating the identical request converges to the same final state (first call succeeds and sets `new_version`; a retry's CAS fails because the state already moved, returning `VERSION_CONFLICT`, but the resource ends up in the same place either way). Matches the `PUT_IF_VERSION_MATCHES` naming precedent from the KV store doc.

Caveat: this API is RPC-style, not pure REST, so a team could reasonably just use `POST` for both instead.

---

## 13. What does a "CAS check" mean?

CAS = **Compare-And-Swap**: a conditional write — "update this value, but only if it still matches what I last saw; otherwise reject."

**Pattern:**
1. Read a record, note its version (e.g. `current_version = 5`).
2. Compute the change locally.
3. Write conditionally: `SET current_version = 6 WHERE id = ? AND current_version = 5`. If someone else already wrote in between, the condition fails and the write is rejected — no silent lost update.

**In this design:** `BeginVersion`/`BeginFileVersion` hands the client `expected_current_version` (the baseline). `CommitVersion`/`CommitFileVersion` does the conditional write — "set `current_version = new_version` only if `current_version` is still `expected_version`." Two devices racing on the same file: only the first commit's CAS succeeds; the second's CAS fails (`VERSION_CONFLICT`) and routes to conflict resolution (conflicted-copy fork) instead of silently overwriting the winner.

---

## 14. When a file is uploaded for the first time, what does `CommitVersion` do?

Not spelled out explicitly in the doc, but following its CAS pattern:

- `BeginVersion` for a brand-new file returns `expected_current_version = 0` (or an equivalent "doesn't exist yet" sentinel) — there's no prior `File` record to baseline against.
- `CommitVersion(file_id, expected_version=0, new_version=1, chunk_hash_list[], ...)` does double duty:
  1. **Creates the `File` record** for the first time (`file_id, owner_id, path, current_version=1, is_deleted=false, created_at, modified_at`).
  2. **Creates `Version 1`** — the first row in `file_versions`, with the full `chunk_hash_list[]`.
- The CAS condition becomes a **create-if-absent guard**: "create only if no `File` row exists yet for this `file_id`" — still protects against the same race (e.g. a retried commit after a lost response would fail CAS the second time, avoiding a duplicate/overwritten `Version 1`).
- Difference from a normal commit: no local chunk reuse is possible on the read-side dedup (no prior version to diff against) — every chunk goes through the full `ChunkExists → GetUploadUrl → PUT` loop, though global cross-user dedup can still save bytes. The Kafka fan-out (`{namespace_id, file_id, new_version}`) fires the same way as any other commit.

---

## 15. Rename `BeginVersion`/`CommitVersion` to `BeginFileVersion`/`CommitFileVersion`

Applied — and this also fixed an existing inconsistency: `GetFileVersion` already used the `FileVersion` naming, so these two were the odd ones out. Updated everywhere: the API table, the §6 Uploader description, the step-by-step upload walk, and the Local↔Remote sequence diagram (realigned the ASCII arrows to preserve column alignment after the longer name).

---

## 16. Add S.No and merge Operation+HTTP Method into Endpoint for the Internal (Service-to-Service) APIs table, same as the Client-Facing table

Applied:

| S.No | Endpoint | Rationale |
|---|---|---|
| 1 | `POST /PublishChangeEvent` | One-way, non-idempotent Kafka publish |
| 2 | `POST /PushNotification` | One-way, non-idempotent socket push |
| 3 | `GET /ResolvePermission` | Pure read, no side effect |
| 4 | `POST /S3ObjectCreated` | One-way S3 event-notification delivery (not "replace a known resource," so not `PUT`) |

---

## 17. Does "Block Storage" in `POST /S3ObjectCreated` mean S3?

Yes. Per the doc's own diagrams (`Client → Chunk Broker → Block Storage (S3)`), **Block Storage = S3** throughout this design — "Block Storage Service" is the logical component name (§9), S3 is the concrete implementation used everywhere (content-addressed object store, prefix-sharded keys like `s3://blocks/{hash[0:2]}/{hash[2:4]}/{full_hash}`). So that table row means: S3 fires an S3 Event Notification when a chunk lands, delivered via SQS/SNS, consumed by the Chunk Broker.

---

## 18. Rename "Block Storage" to "Block Storage (S3)"

Applied across ~26 occurrences throughout the doc (API table notes, architecture diagrams, §9, §12 upload/download flows, §13 dedup, §18 sharding, failure-mode table) for consistency with the places that already said "Block Storage (S3)." Left the section heading (`## 9. Component 3: Block Storage Service`) and its TOC anchor target untouched to avoid breaking the many `#9-component-3-block-storage-service` cross-reference links elsewhere in the doc (TOC *display text* was updated since that's independent of the anchor href). Also skipped the bordered ASCII architecture diagram box where "Block Storage" and "(S3)" already appear on adjacent lines inside the box — appending inline there would've overflowed the box width.

**Side note flagged, not yet fixed:** a leftover inconsistency was spotted near the upload-flow description — "...uploads the chunk DIRECTLY to Block Storage (S3) ... — Block Server is never on the hook for the actual bytes" appears to be stale text from an older draft that used "Block Server" terminology; it should likely read "Chunk Broker is never on the hook..." Also spotted: one old `PATCH /files/{id}` reference in the §12 "Upload (small-to-medium file)" walk that was never updated to the current `CommitFileVersion` endpoint naming. Neither has been fixed yet.

---

## 19. What do SQS and SNS mean?

- **SQS = Simple Queue Service** — AWS's managed point-to-point message queue; one producer pushes, one consumer typically pulls and processes each message.
- **SNS = Simple Notification Service** — AWS's managed pub-sub topic; one publish fans out to every subscriber (which can be multiple SQS queues, Lambda functions, HTTP endpoints, etc.).

**In this design:** when S3 finishes storing an uploaded chunk, it fires an S3 Event Notification, routed either directly to an SQS queue or via an SNS topic fanning out to multiple SQS queues. The Chunk Broker drains that queue and flips the chunk's `status` from `uploading` → `uploaded`. This decouples S3 from the Chunk Broker (durable, retryable delivery) rather than S3 calling the Chunk Broker directly.

---

## 20. 120-second talk: how does the sync agent detect edits and upload them?

> "The client has three pieces working together: a **Watcher**, a **Chunker**, and a local **Indexer**.
>
> The Watcher sits on OS-level filesystem hooks — FSEvents on macOS, inotify on Linux, ReadDirectoryChangesW on Windows — so it's notified of changes, not polling for them. It debounces rapid-fire edits into a single signal.
>
> Once a change settles, the Chunker splits the file using **content-defined chunking** — a rolling hash picks chunk boundaries based on content, not fixed offsets. That matters because if you insert one byte into a file, fixed-size chunking shifts every chunk after it and invalidates all their hashes. CDC means only the chunk actually touched by the edit changes.
>
> Then the upload sequence: the client calls `BeginFileVersion` to get a CAS baseline. For each chunk, it asks `ChunkExists` — a dedup check — and only calls `GetUploadUrl` for chunks the server doesn't already have. Those go straight to S3 via a presigned URL; the Chunk Broker never touches the bytes.
>
> S3 itself fires the upload confirmation — not the client — so a crash right after the PUT still lands correctly.
>
> Finally, `CommitFileVersion` writes the new version with the full chunk list, conditioned on that CAS baseline. If another device won the race, it comes back `VERSION_CONFLICT` and we fork a conflicted copy instead of silently overwriting. The committing device also updates its own local index immediately — it doesn't wait for a round-trip notification to know about its own write."

(~300 words, ≈120 seconds at normal interview pace.)

---

## 21. How does the design handle uploading large files (up to 50GB) given typical POST body size limits?

**Core move: there is no 50GB request.** The file is chunked (~4MB average) before anything touches the network — ~12,800 chunks for 50GB, each uploaded independently.

1. **Small, bounded request bodies.** Each chunk upload is ~4MB, nowhere near typical body-size limits or S3's single-PUT ceiling (5GB). The 50GB file never exists as a single payload anywhere in the pipeline.
2. **The app server never sees the bytes at all.** Presigned URLs mean each chunk PUTs direct from client to S3 — the Chunk Broker only brokers a hash and a URL, so "server request-body limits" become S3's problem, and S3 is built for multi-GB objects.
3. **Resumability falls out for free.** A crash at chunk 8,000 of 12,800 just means re-running `ChunkExists` for every chunk on restart; the ~8,000 already-landed chunks skip, only the rest re-upload.
4. **Parallelism for throughput.** Several chunks upload concurrently, avoiding one giant streaming connection that proxies/load balancers might buffer or time out awkwardly.

**Gotcha worth naming:** CDC produces variable-size chunks, so a hard ceiling (e.g. 8–16MB cap) on top of the ~4MB average target is what actually guarantees no chunk approaches any size limit — not just the average.

---

## 22. How do you prevent a client from committing a file version before all referenced chunks are durably present, given chunks upload independently/in parallel?

**The commit doesn't trust the client's word — it trusts storage's own confirmation, and gates the commit on it.**

1. **Explicit status field.** Every chunk record is `chunk_hash → {size, ref_count, status: uploading | uploaded}`. `status="uploading"` is written before any bytes move.
2. **Only S3 can flip status to `uploaded`.** S3's own Event Notification (via SQS/SNS) triggers the Chunk Broker to verify and flip `status: uploading → uploaded` + bump `ref_count`. The client never makes this call — so it can't claim completion that isn't real, even if crashed or lying.
3. **`CommitFileVersion` checks every referenced chunk's status before accepting the commit.** If any referenced chunk is still `uploading`, the commit is rejected with `CHUNKS_PENDING` instead of accepted; client backs off and retries.

**Race safety:** the status transition is one-directional (`uploading → uploaded`, never back), so the worst case from a race is a false "not ready yet" (costs a retry) — never a false "ready."

**Failure mode covered separately:** a lost (not just delayed) S3 notification would strand a chunk at `uploading` forever. A periodic reconciliation sweep does a `HEAD` on S3 for chunks past their presigned URL's `expires_at`: object present → commit it (replays the missed notification); object absent → delete the stale record so the hash can be retried from scratch.

---

## 23. How does the design let users resume an interrupted upload without starting over?

**Resumability is an emergent property of chunking + dedup, not a separate feature.**

1. **The client doesn't need to remember its own progress.** On restart it just re-runs the normal dedup-check loop: `ChunkExists` for every chunk hash. Already-landed chunks (confirmed `status="uploaded"` from the prior session) report `true` and are skipped — no re-upload, no re-encryption. This works across crashes, app restarts, reboots, even a different process — progress lives server-side in the chunk table, not client memory.
2. **Losing network mid-chunk only costs that one chunk.** Chunks are small (~4MB average, hard-capped), and a plain S3 PUT is atomic — no partial-object corruption to reconcile.
3. **Presigned URL expiry is handled by the normal flow, not special-cased.** If the URL expired during the interruption, the client just calls `GetUploadUrl` again.
4. **The version-commit step is resumable via CAS.** Retrying `CommitFileVersion` after an ambiguous interruption is safe: either it wasn't applied (retry succeeds) or it was applied and the CAS condition now fails (`VERSION_CONFLICT`, so the client re-reads state rather than double-committing).

**Net effect:** no distinct "resumable upload protocol" (upload IDs, part numbers, multipart completion/abort) is needed — it falls out of content-defined chunking + the dedup check already required for other reasons.

---

## 24. How do you prevent one file's upload from accidentally reusing another file's chunks before the first upload is actually ready to commit, while keeping resume behavior?

**Two-layer invariant:**

**Layer 1 — existence ≠ permission to commit (already covered by §22's gate).** Let `ChunkExists` stay optimistic — reports "exists" as soon as any row is present, regardless of `status`. This is desirable: if File A and File B share a chunk and A's upload is in flight, B shouldn't redundantly re-upload identical bytes. Safety comes from `CommitFileVersion` requiring `status="uploaded"` for every chunk, with **no exception for dedup-reused chunks** — if B reused A's in-flight chunk and tries to commit early, B gets `CHUNKS_PENDING`, exactly like A would. Dedup-reuse carries no special trust.

**Layer 2 — what if A's upload never finishes?** Without a fix, B would retry `CommitFileVersion` forever against a chunk stuck at `status="uploading"`. The reconciliation sweep (§22) resolves this generally, not just for the original uploader: once the chunk's row passes its presigned URL's `expires_at` with no object present in S3, the stale record is deleted — resetting the hash to "doesn't exist" for everyone. The next `ChunkExists` (from B, since A gave up) reports `false`, and B does a real upload. Deadlock resolves within the TTL window, not indefinitely.

**Bookkeeping detail this forces:** `ref_count` must increment **per committing Version**, not per physical upload — otherwise B's later successful commit (reusing A's chunk via dedup) would never be counted as a referencer, and GC could delete a chunk B still points to once A's version is later removed.

**Net:** `ChunkExists` stays cheap/optimistic (preserves resume behavior and dedup bandwidth savings); all correctness lives in one uniformly-applied gate (`CommitFileVersion`'s `status="uploaded"` check), backed by the TTL-bound sweep so an abandoned upload can't permanently strand a second file.

---

## 25. How can bandwidth usage be reduced, and sync made faster than re-downloading full files every time they change?

1. **Chunk-level delta sync (the biggest lever).** Content-defined chunking means an edit only invalidates the chunks it touches. Upload skips `ChunkExists`-confirmed-present chunks; download diffs the new version's hash list against the local chunk cache and only pulls what's missing.
2. **Global, cross-user dedup compounds it.** Content-addressed chunks mean identical bytes across users/files are physically one object — the first upload pays the bandwidth, everyone else's `ChunkExists` comes back `true`.
3. **Compress + encrypt once, client-side**, before the chunk ever leaves the device — no uncompressed hop, no server-side reprocessing.
4. **CDN edge caching for "hot" chunks** (shared/popular/high `ref_count`) removes repeat origin trips — first request near an edge is a cache miss (pulls from S3 once), every subsequent nearby request is served from the edge.
5. **The notification layer stays payload-free.** A push to other devices is just `{namespace_id, latest_known_version}` — constant-size regardless of file size — so fan-out cost doesn't scale with what changed.
6. **Metadata delta is itself a delta.** `GetMetadataDelta(since_version_cursor)` returns only what changed since the device's last cursor, not a full re-scan.

**Why this is faster, not just lighter:** smaller transfers mean lower latency, especially on constrained links; dedup/cache hits need no network trip at all; sync latency tracks the *size of the edit*, not the size of the file.
