# Optimistic Locking in Distributed Systems

## The core problem

Two clients read the same record, each plans a write based on what they read, and both try to write back. If nothing coordinates them, the second write silently clobbers the first — the classic **lost update**. In a single-node database you'd reach for a pessimistic lock (`SELECT ... FOR UPDATE`) and hold it for the duration of the read-modify-write. In a distributed system, holding a lock across a network round trip is expensive: it ties up a connection/lock slot for the length of a client think-time, doesn't survive the lock-holder crashing or partitioning away, and doesn't scale to many concurrent readers of mostly-uncontended data.

Optimistic locking sidesteps holding a lock at all. It bets that most concurrent operations on the same record don't actually collide, and only pays a coordination cost when a collision is detected — at commit time, not at read time.

## How it works

Every record carries a **version marker** — a monotonically increasing integer, a vector clock, or a content hash. The read-modify-write cycle looks like:

1. **Read** the record and note its current version, e.g. `version = 5`.
2. **Compute** the new value locally (no lock held, no network call in flight).
3. **Write conditionally**: "update this record to the new value, *but only if* its version is still 5." This is a **Compare-And-Swap (CAS)**, implemented as a conditional write:

```sql
UPDATE file_metadata
SET current_version = new_version, payload = ...
WHERE id = ? AND current_version = expected_version   -- the CAS condition
```

4. **Check the result**:
   - CAS succeeds → no one else wrote in between → commit, bump the version (e.g. to 6).
   - CAS fails → someone else already advanced the version → **this is a genuine conflict**, not an error to retry blindly. The caller must re-read the latest version and either reapply its change, merge, or surface the conflict (e.g. a "conflicted copy" fork), rather than silently overwriting.

No lock is ever held between step 1 and step 3 — the "locking" is really just a conditional write that *detects* a collision after the fact, rather than a mutex that *prevents* one.

### Where the version marker comes from

| Marker | Mechanism | Used by |
|---|---|---|
| Monotonic integer | `current_version` column bumped on every successful write | Relational/metadata stores |
| Vector clock | `{node: counter}` map, compared for causal dominance vs. concurrency | Dynamo-style quorum stores (see [[VectorClocks]]) |
| Content hash / ETag | Hash of the payload itself; any change changes the hash | Object storage, HTTP conditional requests (`If-Match`) |
| Linearizable pointer swap | CAS on a "current version pointer" row, the data itself is immutable | Content-addressed / versioned storage |

## Pros

- **No lock held across a network round trip.** The expensive part (client think-time, network latency between read and write) costs nothing in contention — there's no lock to wait on.
- **Scales well under low contention.** If collisions are rare, almost every write succeeds on the first CAS attempt; throughput approaches the uncontended case.
- **No deadlock, no lock-holder-crashed problem.** There's no lock to leak if a client dies mid-operation between read and write — the next writer just succeeds or fails the CAS on its own.
- **Plays well with horizontal scaling and partition tolerance.** Each write is a self-contained conditional operation; no cross-node lock manager or lease service is required (contrast with pessimistic distributed locks, which need something like a lock service with leases/heartbeats).
- **Conflict detection, not conflict prevention — and that's often the right tradeoff.** The application gets a clear, explicit signal ("your version is stale") instead of a silent lost update.

## Cons

- **Degrades badly under high contention.** If many clients repeatedly collide on the same hot record, they retry over and over (CAS fail → re-read → recompute → CAS again), wasting work and potentially never converging under sustained write pressure — this is where a pessimistic lock would actually do less total work.
- **Pushes conflict handling onto the caller.** A failed CAS is not a transparent retry-and-forget; the application must decide how to reconcile — reapply, merge, or fork a conflict copy. This is extra complexity that a pessimistic lock avoids (because the conflict never arises in the first place).
- **No guarantee of progress for any individual writer.** Under pathological contention, a single client can keep losing the race indefinitely (starvation) — pessimistic locks with fair queuing can guarantee eventual progress; naive optimistic retry loops cannot.
- **Version-marker bookkeeping.** Every record needs a version/clock/hash field, every write path needs to read-then-conditionally-write instead of blind-write, and every caller needs retry logic — more moving parts than "just lock it."
- **Vector-clock variants add "sibling explosion" risk.** If using vector clocks rather than a simple integer, unpruned clocks grow unboundedly with the number of distinct writer nodes over time (see [[VectorClocks]]).

## Optimistic locking vs. pessimistic locking

| | Optimistic locking (CAS) | Pessimistic locking (`SELECT FOR UPDATE`, distributed lease) |
|---|---|---|
| When it coordinates | At commit time, only if a collision actually happened | Up front, before the read, whether or not a collision would have happened |
| Cost under low contention | Near zero | Lock-acquire overhead even when uncontended |
| Cost under high contention | High — repeated retry storms | Bounded — waiters queue instead of burning CPU retrying |
| Failure mode if a holder dies mid-operation | None — there's no lock to leak | Lock must be released via timeout/lease, or everyone else blocks |
| Who resolves a conflict | The application (merge/retry/fork) | The lock manager (conflict never surfaces) |

**Rule of thumb:** choose optimistic locking when writes to the same record are infrequent relative to reads, and conflicts are the exception; choose pessimistic locking when the same record is hammered by concurrent writers and you'd rather queue than retry.

## Where this repo's system designs use it

Grepping the `HighLevelSystemDesign` docs for CAS / conditional-write / version-based concurrency control turns up optimistic locking in:

| Design | Where it's used |
|---|---|
| [`01_DesignKeyValueStore.md`](../HighLevelSystemDesign/01_DesignKeyValueStore.md) | `PUT_IF_VERSION_MATCHES` API — a CAS primitive that rejects a write if the key's current vector clock has diverged from what the client last read |
| [`02_Dropbox_system_design.md`](../HighLevelSystemDesign/02_Dropbox_system_design.md) | CAS on `current_version` in file metadata at commit time, so two devices editing the same file can't both "win"; a failed CAS routes to conflicted-copy conflict resolution instead of a silent overwrite |
| [`01_url_shortener_system_design.md`](../HighLevelSystemDesign/01_url_shortener_system_design.md) | The `URL` record carries a `version` column specifically so edits (`PUT /urls/{short_code}`) go through CAS instead of blind overwrite |
| [`19_Key_vault_system_design.md`](../HighLevelSystemDesign/19_Key_vault_system_design.md) | Secret versions are immutable; the "current version" pointer is moved via a linearizable CAS on the metadata row, so two writers can't both believe they hold the current version |
| [`18_RBAC_ABAC_merchandising_catalog_system_design.md`](../HighLevelSystemDesign/18_RBAC_ABAC_merchandising_catalog_system_design.md) | Inventory reservation/decrement path uses optimistic concurrency plus idempotency keys, rather than locking the inventory row for the duration of a checkout |
| [`00_UniqueIDGenerator.md`](../HighLevelSystemDesign/00_UniqueIDGenerator.md) | Worker-ID claiming via `SELECT ... FOR UPDATE` / CAS on a `worker_ids` table as a row-based alternative to ZooKeeper leases |
| [`03_WebCrawler.md`](../HighLevelSystemDesign/03_WebCrawler.md) | New-domain race condition fix: CAS semantics on the domain map (alternative to a Redis `SET NX` distributed lock) to ensure only one fetcher creates the back queue for a newly discovered domain |
