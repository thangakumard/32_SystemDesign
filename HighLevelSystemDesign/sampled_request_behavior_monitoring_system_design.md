# Sampled Request Behavior Monitoring — System Design Interview Guide
### Senior & Staff Engineer Level

---

## Table of Contents

1. [Problem Statement & Scope](#1-problem-statement--scope)
2. [Requirements](#2-requirements)
3. [Capacity Estimation](#3-capacity-estimation)
4. [High-Level Architecture](#4-high-level-architecture)
5. [Component 1: Sampling Interceptor](#5-component-1-sampling-interceptor)
6. [Component 2: Async Capture & PII Redaction](#6-component-2-async-capture--pii-redaction)
7. [Component 3: Transport Layer (Kafka)](#7-component-3-transport-layer-kafka)
8. [Component 4: Replay Dispatcher](#8-component-4-replay-dispatcher)
9. [Component 5: Side-Effect Isolation Layer](#9-component-5-side-effect-isolation-layer)
10. [Component 6: Response Correlation & Join](#10-component-6-response-correlation--join)
11. [Component 7: Diff Engine](#11-component-7-diff-engine)
12. [Component 8: Noise-Floor Baseline (Dual-Primary Pattern)](#12-component-8-noise-floor-baseline-dual-primary-pattern)
13. [Component 9: Diff Aggregation, Storage & Release Gate](#13-component-9-diff-aggregation-storage--release-gate)
14. [Sampling Strategy Deep Dive](#14-sampling-strategy-deep-dive)
15. [Alternative Architectures & Trade-offs](#15-alternative-architectures--trade-offs)
16. [CAP / PACELC Positioning](#16-cap--pacelc-positioning)
17. [Failure Modes & Mitigations](#17-failure-modes--mitigations)
18. [Scalability & Sharding](#18-scalability--sharding)
19. [Senior vs Staff Answer Differentiators](#19-senior-vs-staff-answer-differentiators)
20. [Interview Time Allocation](#20-interview-time-allocation)
21. [Quick-Reference Cheatsheet](#21-quick-reference-cheatsheet)
22. [Related Guides](#22-related-guides)

---

## 1. Problem Statement & Scope

Design a **shadow (differential) testing pipeline** for a high-volume API Gateway: sample a slice of real production requests, replay them against a **candidate build** that is not yet serving live traffic, and automatically surface behavioral differences (status codes, response bodies, headers, latency) between the current production build and the candidate — before the candidate is promoted.

### The framing trap to avoid out loud

This is **not** a canary release / A-B rollout question. In a canary, some real users' *responses* are actually served by the candidate — the candidate is in the live serving path. Here the constraint is explicit: *"must not affect the response returned to the production caller."* That single sentence rules out canarying, weighted routing, and feature-flag-gated live traffic splits as the core mechanism. The candidate only ever sees a **copy** of a request; its response goes nowhere except into a diff.

Stating this distinction unprompted — and explaining *why* it's disqualified by the requirements rather than just picking a different design — is itself a Staff-level signal.

### What the interviewer is really testing

- Can you build a **zero-risk-to-production** side channel out of a live, high-QPS path?
- Do you understand that **replaying mutating requests is dangerous** and design explicit side-effect isolation, not just "mirror the traffic"?
- Can you separate **real regressions from inherent non-determinism** (timestamps, IDs, ordering) rather than drowning the signal in noise?
- Do you reason about **statistical validity** — how much sampled traffic is actually enough to trust a "no diffs" result?

### Prior art worth naming

- **Twitter Diffy** — runs Primary, a cloned Primary′, and a Candidate side by side; diffs Primary vs Primary′ to learn the *noise floor* (nondeterminism between two identical instances), then subtracts that noise from Primary vs Candidate. This guide's noise-floor component is directly this pattern.
- **GitHub Scientist** — a related but different technique: an **inline** experiment where both code paths run *in the same process/request*, and the control's result is always what's returned. That solves "does this refactor change behavior?" for a single deployable. It does not fit this prompt, because the prompt specifies a separate **candidate build** — a distinct deployment — not a branch inside one binary. Calling out this distinction shows you understand the design space, not just one point in it.

---

## 2. Requirements

### Functional Requirements

| Requirement | Notes |
|---|---|
| Sample a configurable % of live requests at the Gateway | Uniform + stratified per endpoint |
| Capture request and production response without altering them | Read-only tap, not an interceptor of the response |
| Replay captured requests against a candidate build | Candidate runs in an isolated shadow environment |
| Prevent replay from causing real side effects | Payments, emails, writes, third-party calls must not fire twice |
| Diff candidate vs. production responses | Status, headers (allow-listed), body |
| Separate real regressions from inherent noise | Timestamps, IDs, field ordering, floats |
| Aggregate diffs per endpoint and expose a gating signal | Feeds the release/promotion decision |
| Support multiple candidate builds under test concurrently | Common during a multi-stage rollout |

### Non-Functional Requirements

| Requirement | Target |
|---|---|
| Added latency to the production request path | ~0ms — sampling decision + enqueue only, fully async |
| Added risk to production correctness | Zero — capture path must fail open and never throw into the caller's request |
| Scale | Gateway-independent; sampling infra scales with sample volume, not full traffic |
| Data privacy | PII redacted/tokenized before any persistence or cross-team visibility |
| Statistical confidence | Enough samples per endpoint to detect a target regression rate at a stated confidence level |
| Isolation blast radius | A bug in the shadow pipeline must never be visible to, or throttle, production |

---

## 3. Capacity Estimation

```
Assume: API Gateway peak = 50,000 RPS
Avg captured payload (request + response, pre-redaction): ~10 KB

Baseline uniform sample rate: 0.5% (1-in-200)
  50,000 × 0.005 ≈ 250 req/s from uniform sampling

Stratified floor: guarantee ≥5 req/s replay volume per endpoint
  ~300 distinct endpoints × 5 req/s (worst case, all long-tail) = 1,500 req/s ceiling
  Realistic blended volume (uniform dominates for high-traffic endpoints,
  floor only kicks in for the long tail): ~500–800 req/s sustained

Capture bandwidth into Kafka:
  800 req/s × 10 KB ≈ 8 MB/s raw → ~2.5–3 MB/s after gzip (~3× reduction)
  Trivial for a Kafka cluster already carrying the gateway's own telemetry.

Replay dispatcher fleet:
  800 req/s × ~200ms avg candidate latency ≈ 160 concurrent in-flight replays
  At ~200 connections/node → 1–2 dispatcher nodes (plus headroom) → 4 nodes.
  This fleet is orders of magnitude smaller than the gateway fleet itself —
  a useful thing to say out loud: the shadow pipeline should never be a
  capacity driver.

Diff engine cost:
  Structural JSON diff ≈ 1–5ms CPU per comparison
  800 diffs/s × 3ms avg ≈ 2.4 CPU-seconds/s → ~3 cores steady-state,
  scaled out via partitioned stream processing.

Retention (7-day rolling corpus for regression replay):
  8 MB/s × 86,400s × 7 ≈ 4.8 TB/week raw → ~1.5–2 TB/week compressed
  → object store with lifecycle policy, not a hot database.

Statistical significance ("Rule of Three"):
  To be 95% confident a true regression rate p has NOT occurred after
  observing zero diffs in n samples: n ≈ 3 / p
  Detecting p = 0.5% (1-in-200 requests behave differently) needs
  n ≈ 3 / 0.005 = 600 samples per endpoint before "no diffs seen" is
  actually a meaningful signal — not just "we didn't sample enough."
```

**Key insight:** the shadow pipeline's infrastructure footprint is small relative to the gateway. The hard part isn't throughput — it's **correctness under replay** (no double side effects) and **signal quality** (noise vs. real regression), not raw scale.

---

## 4. High-Level Architecture

```
                         ┌──────────────────────┐
                         │   Client / Caller     │
                         └───────────┬────────────┘
                                     │  single request
                         ┌───────────▼────────────┐
                         │      API Gateway        │
                         │   (production build)    │
                         └───────────┬────────────┘
                                     │
                    ┌────────────────┼──────────────────────┐
           (sync — unaffected)                    (async, fire-and-forget)
                    │                                        │
        ┌───────────▼───────────┐               ┌────────────▼─────────────┐
        │  Production handling   │               │  Sampling Interceptor    │
        │  (unchanged, blocking) │               │  hash(request_id) < p ?  │
        └───────────┬───────────┘               └────────────┬─────────────┘
                    │  response                                │ sampled req + resp
                    ▼                                          ▼
              back to caller                       ┌───────────────────────┐
                                                    │  Capture & Redact     │
                                                    │  (PII scrub, bounded  │
                                                    │   in-proc queue, drop │
                                                    │   on backpressure)    │
                                                    └───────────┬───────────┘
                                                                │
                                                    ┌───────────▼───────────┐
                                                    │  Kafka: sampled-      │
                                                    │  requests / -prod-    │
                                                    │  responses topics     │
                                                    └───────────┬───────────┘
                                                                │
                          ┌─────────────────────────────────────┼─────────────────────────────┐
                          │                                                                     │
             ┌────────────▼─────────────┐                                          ┌────────────▼─────────────┐
             │   Replay Dispatcher       │                                          │   Replay Dispatcher       │
             │   → Candidate Build       │                                          │   → Primary′ (twin)       │
             │   (shadow-mode context)   │                                          │   (noise-floor baseline)  │
             └────────────┬─────────────┘                                          └────────────┬─────────────┘
                          │ candidate response                                                    │ twin response
                          └───────────────────────────────┬────────────────────────────────────────┘
                                                            ▼
                                                ┌────────────────────────┐
                                                │  Correlation & Join     │
                                                │  keyed by request_id,   │
                                                │  bounded time window    │
                                                └────────────┬────────────┘
                                                             ▼
                                                ┌────────────────────────┐
                                                │  Diff Engine            │
                                                │  normalize → structural │
                                                │  diff → subtract noise  │
                                                │  floor (A vs A′)        │
                                                └────────────┬────────────┘
                                                             ▼
                                                ┌────────────────────────┐
                                                │  Aggregation & Storage  │
                                                │  per-endpoint diff rate │
                                                └────────────┬────────────┘
                                                             ▼
                                                ┌────────────────────────┐
                                                │  Dashboard / Release    │
                                                │  Gate (promote/block)   │
                                                └────────────────────────┘
```

**Kafka** decouples every stage exactly as in a crawl pipeline: it lets the diff engine fall behind under load without touching the gateway, gives replay capability against a fresh candidate without re-capturing traffic, and provides the backpressure signal that lets the sampling rate self-throttle.

---

## 5. Component 1: Sampling Interceptor

A lightweight filter/middleware in the Gateway's request pipeline. Its **only** job on the hot path is a cheap sampling decision — everything else happens after the production response has already been sent.

```
Decision (O(1), no I/O):
  sampled = hash(request_id) mod 10_000 < sample_rate_bps
```

Using a hash of a stable identifier (request ID, or user ID for consistent per-user sampling) rather than `rand()` matters for two reasons:
- **Reproducibility** — the same logical request always sampled the same way if retried.
- **Cross-service consistency** — if this Gateway fronts a fan-out to other services that also want to shadow-test in a correlated way, the same hash can gate sampling everywhere without coordination.

The interceptor never blocks on the sampling *outcome path* — see Component 2.

---

## 6. Component 2: Async Capture & PII Redaction

Once a request is marked sampled, the interceptor:

1. Copies the request (headers, body, query params) and — **after** the production response is already written to the caller — copies the response too.
2. Pushes both onto a **bounded in-process queue** (ring buffer). If the queue is full, **drop the sample** — never block, never grow unboundedly, never throw.
3. A background worker drains the queue, redacts/tokenizes known PII fields (auth tokens, payment instruments, emails — a configurable field catalog, not a manual per-endpoint decision) and serializes to Kafka.

```
Bulkhead rule: capture path failures are swallowed and counted,
never propagated. A crash in the redaction library must not become
a 500 for the production caller.
```

This is the first of several places in this design where the answer must explicitly say "fail open, drop the sample" rather than "retry" or "block" — retrying or blocking on the hot path is exactly the failure mode the requirement is designed to catch you on.

---

## 7. Component 3: Transport Layer (Kafka)

Two logical topics (can be one topic with an event-type field, but separating them clarifies retention and consumer groups):

| Topic | Producer | Contents | Partition key |
|---|---|---|---|
| `sampled-requests` | Gateway capture worker | Request only, immediately after capture | `hash(request_id)` |
| `sampled-prod-responses` | Gateway capture worker | Production response, once available | `hash(request_id)` |

Partitioning both topics by the **same key** (`request_id`) means the eventual join in Component 6 can be done with local, partition-affine state rather than a cross-partition shuffle — the same domain-locality argument as routing crawler URLs by domain hash.

Multiple **consumer groups** on `sampled-requests` allow multiple candidate builds to be tested concurrently off the same captured stream, at zero extra capture cost — each candidate's Replay Dispatcher is just another consumer group.

---

## 8. Component 4: Replay Dispatcher

A stateless worker pool that consumes `sampled-requests` and re-issues each request against the candidate build's endpoint.

```
Per message:
  1. Deserialize captured request
  2. Attach shadow-mode context (see Component 5)
  3. Issue to candidate with a bounded timeout (do not retry indefinitely —
     a hung candidate is itself a signal, not a transient error to hide)
  4. Publish candidate response to `candidate-responses` topic, keyed by
     the same request_id
  5. On failure/timeout: publish a synthetic "no-response" diff record —
     a missing response is a meaningful outcome, not a thing to discard
```

Dispatchers are trivially horizontally scalable (stateless, one consumer group per candidate) and are intentionally decoupled in time from the original request — a short, bounded replay delay (seconds) is acceptable and even useful, since it smooths bursts without needing the candidate to absorb the gateway's exact peak QPS.

---

## 9. Component 5: Side-Effect Isolation Layer

**This is the component most candidates skip, and the one that actually determines whether the design is safe to run.** "Mirror the traffic" is not itself an answer — mirrored *mutating* traffic against a live candidate will double-charge customers, send duplicate emails, and write duplicate rows unless every side-effecting boundary understands it's in a replay.

### Shadow-context propagation

Every replayed request carries a marker (e.g., `X-Shadow-Request: true`, plus the original `request_id`) that propagates through the candidate's call graph the same way a trace context propagates — via a context object threaded through downstream clients, not via a global flag.

### Boundary behavior by dependency type

| Dependency | Shadow-mode behavior |
|---|---|
| Primary datastore writes | Redirected to a shadow schema/tenant, or executed as a dry-run that computes but does not commit |
| Primary datastore reads | Passed through to a read replica — safe, and avoids extra load on the primary |
| Payment / billing gateway | Routed to the provider's sandbox endpoint, or stubbed to return a synthetic success/failure matching the original outcome shape |
| Email / SMS / push notifications | Hard no-op — never actually send; log "would have sent" for diffing purposes |
| Calls to other production microservices | Either those services also honor the shadow-mode header (best case, requires org-wide adoption), or the candidate's dependency graph is pointed at stubs/doubles for that dependency |

### The scoping decision this forces

Not every endpoint is safe to shadow-test with the same rigor. A defensible design explicitly partitions endpoints into:
- **Freely shadow-safe** (idempotent GETs, reads) — mirror with no special handling.
- **Shadow-safe with isolation** (writes where the isolation layer above is proven correct) — mirror, but only once isolation is contract-tested.
- **Excluded from replay** (irreversible external effects where the org isn't confident in isolation — e.g., an endpoint that triggers a same-day funds transfer) — deliberately not sampled at all, or sampled only for read-only diffing of *inputs and routing decisions*, not full execution.

Stating this partition explicitly — rather than implying "we mirror everything, isolation solves it" — is a Staff-level tell: it acknowledges that isolation has a trust boundary and that boundary should be enforced by policy, not just hoped for.

---

## 10. Component 6: Response Correlation & Join

A windowed join keyed by `request_id`, combining three streams: production response, candidate response, and (if using the noise-floor pattern) the twin's response.

```
Join store: Redis (or a Kafka Streams / Flink state store), TTL'd
  key:   request_id
  value: { prod_response?, candidate_response?, twin_response? }

On each arrival: merge into the entry; when all expected parts are
present OR the join window expires (e.g., 30s) → emit for diffing.

Partial arrival at window expiry → emit as "incomplete" outcome,
not silently dropped. A candidate that consistently fails to
respond within the window is itself the finding.
```

Because both topics are partitioned by `request_id`, this join can largely be done with local partition state rather than a full shuffle — same locality argument used for the crawler's domain-based routing.

---

## 11. Component 7: Diff Engine

### Normalization before comparison

Comparing raw bytes will drown every real regression in false positives. Normalize first:

```
- Strip or template known non-deterministic fields:
    timestamps, freshly generated request/trace IDs, random UUIDs,
    "processing time" headers, retry counters
- Canonicalize JSON: sort object keys, treat semantically-unordered
  arrays as sets where the schema says order doesn't matter
- Float tolerance: compare within an epsilon, not bit-for-bit
- Header allow-list: only compare headers that are part of the
  documented contract (ignore Date, X-Request-Id, Server, etc.)
```

### Structural, not textual, diff

Diff the normalized payload as a tree (JSON/proto structure), not as a string. This distinguishes "a field was added," "a field's type changed," "a field is null vs. missing" — categories that matter very differently for a behavioral regression than a raw text diff conveys.

### Output

A diff record: `{endpoint, request_id, diff_type, changed_fields[], severity}` — fed into aggregation.

---

## 12. Component 8: Noise-Floor Baseline (Dual-Primary Pattern)

Even two instances of the **exact same production build** will not produce byte-identical responses under load — cache warmth, timing-sensitive fields not yet normalized, load-balancer-dependent routing metadata, etc. Comparing production directly against the candidate without accounting for this baseline will misattribute inherent noise to the candidate.

**Pattern (after Twitter's Diffy):** run a **Primary′** — a second instance of the *current production build*, receiving the same replayed traffic as the candidate — purely to measure Primary vs. Primary′ diffs. Any diff pattern that shows up there is noise by definition, since both sides are running identical code. Subtract that noise pattern (by diff fingerprint, see Component 9) from the Primary vs. Candidate diff stream before it reaches a human or a gate.

```
Primary  vs  Primary′   →  noise floor (same code, different instance)
Primary  vs  Candidate  →  raw diffs (candidate code changes + noise)

Reported regression signal = (Primary vs Candidate diffs)
                              minus (diff fingerprints also seen in
                                     Primary vs Primary′)
```

This is the single highest-leverage idea in this design, and the one that separates a system that's actually trustworthy from one that generates so much noise nobody looks at it.

---

## 13. Component 9: Diff Aggregation, Storage & Release Gate

### Fingerprinting to fight alert fatigue

Group diffs by a **fingerprint of the diff shape** (which fields changed, not their exact values) rather than by raw content — this turns "10,000 diffs" into "3 distinct diff patterns, occurring at these rates," which is what a human (or a gate) can actually act on.

```sql
CREATE TABLE diff_aggregates (
  endpoint         TEXT,
  diff_fingerprint TEXT,
  candidate_build  TEXT,
  first_seen       TIMESTAMP,
  occurrence_count BIGINT,
  sample_count     BIGINT,
  is_noise         BOOLEAN,     -- also seen in Primary vs Primary′
  PRIMARY KEY (endpoint, candidate_build, diff_fingerprint)
);
```

### Release gate policy

```
For each endpoint, given n samples and the Rule-of-Three target
regression rate p_target:
  IF n < 3 / p_target  →  "insufficient samples" (do not gate on it yet)
  ELSE IF non-noise diff rate > p_target  →  BLOCK promotion
  ELSE  →  PASS
```

Surfacing "insufficient samples" as a distinct state — rather than silently treating low-traffic endpoints as "passed" — is what keeps this system honest for the long tail of rarely-hit endpoints.

---

## 14. Sampling Strategy Deep Dive

| Strategy | What it does | Trade-off |
|---|---|---|
| **Uniform** | Fixed % across all traffic | Simple, statistically clean, but under-samples rare endpoints |
| **Stratified** | Per-endpoint minimum floor (e.g. ≥5 req/s) | Guarantees coverage of the long tail; small added volume |
| **Guided / biased** | Over-sample traffic likely to be interesting — prior error responses, edge-case parameter shapes, previously-diffing request patterns | Finds bugs faster, but must be tracked in a **separate stream** from the uniform one — mixing them breaks the statistical validity of the gate's diff-rate math |

Report guided-sample findings as "bugs found," and uniform-sample findings as "the statistically valid diff rate used for gating." Conflating the two is a subtle correctness bug in the monitoring system itself.

---

## 15. Alternative Architectures & Trade-offs

| Approach | Description | Pros | Cons |
|---|---|---|---|
| **A. App-level async mirror (this guide's primary design)** | Gateway interceptor samples, captures, and asynchronously replays via a queue | Fine-grained sampling control (per-endpoint, per-user, consistent hashing); easy to inject shadow-mode context; captures the real production response for comparison | Requires code in the gateway's request pipeline; small per-request overhead (bounded, async) |
| **B. Infra-level traffic mirroring** (e.g., Envoy/Istio `request_mirror_policy`) | Service mesh duplicates the raw request to a shadow cluster at the proxy layer | Zero application code change; battle-tested; guarantees the primary path is truly untouched since mirroring happens outside app code | Mirror response is normally discarded, so capturing the *production* response for diffing needs a separate correlation mechanism; little control over consistent/stratified sampling or shadow-context injection without a custom filter |
| **C. Record-then-replay (offline corpus)** | Persist sampled requests to durable storage; replay in batch against a candidate later, decoupled in time | Fully decoupled from live timing; can replay the same corpus against multiple candidate builds for repeatable regression testing; simplest to reason about blast radius | Data/state staleness — the DB state a replayed request expects may have moved on, producing false diffs unrelated to code; slower feedback loop |
| **D. Synthetic canary probes** | Periodically send hand-crafted requests to the candidate, not real traffic | Simple, no isolation concerns for read probes | Explicitly fails the prompt's premise — it doesn't use *sampled production requests*, so it inherits the same blind spots as unit/integration tests: it can't cover the diversity of real payload shapes and client behavior |
| **E. Real canary release** | Route a small % of live traffic's actual serving to the candidate | Tests the real deployed artifact under real load, including infra-level concerns | Disqualified by the requirement: live users' responses would come from the candidate, violating "must not affect the response returned to the production caller" |

**Recommended answer:** primary design is **A**, optionally backed by **C** for a durable regression corpus that can be replayed against future candidates without re-capturing traffic, with **B** mentioned as a legitimate lower-effort alternative when application-level control over sampling/isolation isn't required (e.g., early-stage validation of read-only endpoints).

---

## 16. CAP / PACELC Positioning

| Component | Position | Reasoning |
|---|---|---|
| `sampled-requests` / `sampled-prod-responses` (Kafka) | **AP** | Losing a sample means one fewer data point, not incorrect production behavior. At-least-once delivery is fine — duplicate diff evaluation is harmless. |
| Correlation/join store (Redis) | **AP** | An expired or lost join entry becomes an "incomplete" outcome — acceptable, not correctness-critical. |
| Diff aggregate store (Cassandra/ClickHouse) | **AP / EL** | Dashboards and gates can tolerate eventual consistency; favor low write latency at high sample volume. |
| Sampling-rate config plane | **CP-leaning** | All gateway nodes should apply a consistent policy; brief staleness (seconds) is tolerable, but a split-brain where half the fleet samples at 0.5% and half at 50% is a real operational risk. |
| Side-effect isolation contract (shadow-mode propagation) | **Correctness-critical, not a CAP axis** | This isn't a data-store trade-off — it's a hard invariant. There is no "eventually isolated." |

---

## 17. Failure Modes & Mitigations

| Failure | Symptom | Mitigation |
|---|---|---|
| Capture path throws or blocks | Latency spike / errors on production requests | Bulkhead: capture runs in a bounded, non-blocking queue; drop-on-full; never propagate exceptions to the caller |
| Side-effect leakage (candidate performs a real mutation/3rd-party call) | Duplicate charges, duplicate emails, dirty prod data | Mandatory shadow-context propagation; contract tests that assert isolation before an endpoint is enabled for replay; a kill switch per endpoint |
| Diff noise flood | Real regressions buried under thousands of cosmetic diffs | Normalization + structural diff + dual-primary noise-floor subtraction; fingerprint-based aggregation |
| Insufficient sampling on low-traffic endpoints | False "all clear" on rarely-hit paths | Stratified sampling floor; explicit "insufficient samples" gate state (Rule of Three) |
| Data/state staleness in offline replay | False diffs from a candidate reading state that has since changed | Bound replay delay to seconds/minutes for the live-mirror path; for the offline corpus, normalize state-dependent fields or accept some inherent noise there |
| Correlation join never completes | Candidate is slow/hung; diff evaluation never fires | Bounded join window; emit "no-response" as a first-class diff outcome, not a silent drop |
| Kafka lag on the sampled-traffic topic | Shadow pipeline processes stale traffic | Monitor consumer lag; auto-reduce sampling rate under lag rather than let staleness grow unbounded — freshness matters more than volume here |
| The diffing pipeline itself has a bug (false negative) | Real regressions silently pass through undetected | Periodically inject a **known-bad build** as a chaos test of the safety net itself — verify the pipeline actually flags a deliberately broken candidate |
| PII exposure in captured payloads | Compliance/privacy incident | Redact/tokenize at capture time, before persistence; restrict diff-store access; TTL-based deletion |
| Sandbox drift (payment/3rd-party sandbox behaves differently than production) | False diffs unrelated to the candidate's code | Track sandbox-attributable diff fingerprints separately; don't let sandbox noise block promotion |

---

## 18. Scalability & Sharding

```
Kafka partitioning:
  Both sampled-requests and sampled-prod-responses partitioned by
  hash(request_id) → co-locates the pair needed for the eventual join,
  the same domain-locality principle used for crawler URL routing.

Multi-candidate support:
  Each candidate build's Replay Dispatcher is a distinct consumer
  group on sampled-requests. Kafka's pub/sub fan-out means testing
  N candidates concurrently costs zero extra capture overhead —
  only N× replay/diff compute, which is cheap relative to the gateway.

Diff engine parallelism:
  Stream processing (Kafka Streams / Flink) task count scales with
  partition count; state (join buffers) partitioned the same way.

Sampling-rate distribution:
  Computed locally per gateway node from a periodically-refreshed,
  locally cached policy — no per-request coordination call. Avoids
  a config service becoming a hot-path dependency.
```

---

## 19. Senior vs Staff Answer Differentiators

### Senior-level expectations

- Identify the need to sample and asynchronously mirror traffic
- Mention comparing candidate vs. production responses
- Recognize that replay must not add latency to the caller
- Basic capacity math on sample volume

### Staff-level expectations (additional)

| Topic | What Staff-level adds |
|---|---|
| Framing | Explicitly rules out real canary release using the "must not affect the caller's response" constraint, and explains why |
| Side effects | Names the isolation layer as its own component with per-dependency-type behavior, not an assumed side effect of "shadow mode" |
| Noise | Dual-primary (Diffy-style) noise-floor subtraction, not just "ignore timestamps" |
| Sampling | Stratified + guided sampling kept in separate statistical streams |
| Statistical rigor | Rule-of-Three minimum sample size before "no diffs" is a trustworthy signal |
| Failure handling | Chaos-tests the safety net itself with a known-bad build |
| Scoping | Explicitly partitions endpoints into freely-safe / isolation-required / excluded, rather than claiming universal safety |
| Multi-build support | Multiple candidates via Kafka consumer-group fan-out at no extra capture cost |
| Alternatives | Compares app-level mirroring, infra-level (Envoy) mirroring, and offline record/replay with honest trade-offs rather than presenting one design as the only option |

### The single most important Staff differentiator

**Treating side-effect isolation as a first-class, explicitly-scoped component** rather than a hand-wave. Anyone can say "mirror the traffic asynchronously." The design that's actually safe to run in production is the one that says: here is exactly which dependency types get stubbed, redirected, or excluded, and here is the boundary beyond which we deliberately don't replay.

---

## 20. Interview Time Allocation

For a 45-minute system design session:

| Phase | Time | Focus |
|---|---|---|
| Requirements & scope | 5 min | Confirm "must not affect production response"; rule out canary as the mechanism |
| Capacity estimation | 4 min | Sample volume, Kafka bandwidth, Rule-of-Three sample-size math |
| High-level architecture | 5 min | Draw the async capture → Kafka → replay → diff pipeline |
| Side-effect isolation (deep dive) | 10 min | Shadow-context propagation, per-dependency behavior, endpoint scoping |
| Diffing & noise floor | 10 min | Normalization, structural diff, dual-primary pattern |
| Sampling strategy | 4 min | Uniform vs. stratified vs. guided, kept statistically separate |
| Failure modes | 5 min | Capture-path bulkhead, side-effect leakage, noise flood, chaos-test the pipeline |
| Alternatives (if time) | 2 min | Infra-level mirroring vs. offline record/replay trade-offs |

**Never cut:** side-effect isolation and the noise-floor pattern — these are the two ideas that separate a real design from "mirror the traffic and diff it."

---

## 21. Quick-Reference Cheatsheet

```
CORE PRINCIPLE
──────────────
Candidate NEVER influences the caller's response. It only ever
receives a copy, asynchronously, after the fact.

KEY COMPONENTS
──────────────
Sampling:        hash(request_id) mod N < rate  → consistent, O(1), no I/O
Capture:         async, bounded queue, drop-on-full, never blocks caller
Transport:       Kafka, partitioned by request_id (join locality)
Isolation:       shadow-context propagation; per-dependency-type stubbing
Diffing:         normalize → structural diff → subtract noise floor
Noise floor:     Primary vs Primary′ (Diffy pattern) — same code, different instance
Gating:          Rule of Three → n ≥ 3/p_target before "no diffs" is trustworthy

KEY NUMBERS (illustrative)
───────────────────────────
50,000 RPS gateway → ~500–800 req/s sampled (uniform + stratified floor)
~3 MB/s compressed capture bandwidth into Kafka
n ≈ 3 / p_target samples needed per endpoint for a given confidence

ENDPOINT SCOPING
─────────────────
Freely shadow-safe:      idempotent reads
Shadow-safe w/ isolation: writes, once isolation is contract-tested
Excluded from replay:    irreversible external effects, low isolation confidence

CAP DECISIONS
─────────────
AP:  capture topics, join store, diff aggregate store
CP-leaning: sampling-rate config plane
Not a CAP axis: side-effect isolation — it's a hard correctness invariant

STAFF-LEVEL SIGNALS TO HIT
───────────────────────────
✓ Explicitly rules out real canary release and explains why
✓ Side-effect isolation as its own scoped component, not hand-waved
✓ Dual-primary noise-floor subtraction (Diffy pattern)
✓ Rule-of-Three statistical sample-size reasoning
✓ Separate statistical streams for guided vs. uniform sampling
✓ Chaos-tests the diffing pipeline itself with a known-bad build
✓ Multi-candidate support via consumer-group fan-out, no extra capture cost
✓ Honest trade-off table across app-level, infra-level, and offline approaches
```

---

## 22. Related Guides

- **Apache Kafka** — partitioning, consumer groups, and replay/back-pressure mechanics underlying the transport layer here.
- **Circuit Breaker Pattern** — the bulkhead/fail-open reasoning behind the capture path never blocking the caller.
- **CAP/PACELC Theorem** — the general framework applied per-component in Section 16.
- **Distributed Unique ID Generation** — `request_id` as the correlation key threading capture, replay, and join.
- **Sharding Strategies** — the partition-by-key-for-join-locality pattern reused from domain-hash routing.
- **Endpoint Telemetry Pipeline** — a closely related sampled-ingestion architecture; contrast telemetry (observe and aggregate) vs. this system (observe, replay, and diff against a second execution).
- **VirusTotal-style Multi-Engine Scanning Platform** — another design centered on comparing multiple engines/executions against the same input and reconciling disagreement.

---

*Guide compiled for Senior & Staff-level system design interview preparation.*
*Topics: Shadow Testing, Differential Testing, Traffic Mirroring, Side-Effect Isolation, Release Gating.*
