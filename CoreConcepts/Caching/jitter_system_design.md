# Jitter — System Design Interview Guide
### Senior & Staff Engineer Level

---

## Table of Contents

1. [Definition: Two Meanings of Jitter](#1-definition-two-meanings-of-jitter)
2. [The Problem: Accidental Synchronization](#2-the-problem-accidental-synchronization)
3. [Jitter Strategies for Retry Backoff](#3-jitter-strategies-for-retry-backoff)
4. [Where Jitter Applies Beyond Retries](#4-where-jitter-applies-beyond-retries)
5. [Applied: Jitter in the Web Crawler](#5-applied-jitter-in-the-web-crawler)
6. [Alternatives & Complements](#6-alternatives--complements)
7. [Failure Modes & Mitigations](#7-failure-modes--mitigations)
8. [Senior vs Staff Answer Differentiators](#8-senior-vs-staff-answer-differentiators)
9. [Quick-Reference Cheatsheet](#9-quick-reference-cheatsheet)
10. [Related Guides](#10-related-guides)

---

## 1. Definition: Two Meanings of Jitter

| Meaning | What it is | Relationship to design |
|---|---|---|
| **Network jitter** | Variation in latency between packets/requests (p50 = 20ms, but individual requests swing 5ms–200ms) | Something you **measure and tolerate**. Matters for VoIP, video, gaming, tail-latency SLOs |
| **Jitter as a technique** | Deliberately adding randomness to timing so independent actors don't act at the same moment | Something you **design in**. The usual meaning in distributed systems interviews |

The rest of this guide covers the second meaning.

**One-line interview definition:**

> Jitter is intentional randomness in timing that desynchronizes independent clients, turning correlated load spikes into a smooth distribution. It is essential for retries, cache expiry and scheduled work. It spreads load but does not reduce it, so pair it with budgets, coalescing and circuit breakers.

---

## 2. The Problem: Accidental Synchronization

Distributed systems synchronize by accident. Whenever many clients share a trigger, they fire together.

| Scenario | What happens | Name |
|---|---|---|
| A dependency blips | 10,000 clients fail at the same instant; with plain exponential backoff they all retry at 1s, 2s, 4s, 8s. Each wave re-crashes the recovering service | **Retry storm** |
| A popular cache key expires | Every reader misses at once and hits the database | **Cache stampede / thundering herd** |
| Every node runs cron at `0 * * * *` | Load spike on the hour | **Scheduled-job spike** |
| All pods start together after a deploy | Heartbeats, token refreshes and cache TTLs stay aligned for the life of the pods | **Deploy-time phase lock** |

**Exponential backoff alone does not fix this.** It spaces retries out but keeps clients **in lockstep**. Jitter breaks the correlation.

```
Without jitter:     ████          ████              ████         (synchronized waves)
With full jitter:   ▂▃▂▃▂▃▂▃▂▃▂▃▂▃▂▃▂▃▂▃▂▃▂▃▂▃▂▃▂▃▂▃▂▃▂▃▂▃▂▃     (smoothed load)
```

**Key insight:** jitter does not reduce total work. It converts a spike into a plateau. Systems usually fail on **peak** load, not average load.

---

## 3. Jitter Strategies for Retry Backoff

Notation: `base` = initial delay, `cap` = maximum delay, `n` = attempt number.
(Analysis popularized by Marc Brooker, AWS Architecture Blog.)

| Strategy | Formula | Pros | Cons |
|---|---|---|---|
| **No jitter** | `min(cap, base·2ⁿ)` | Predictable, easy to reason about | Clients stay synchronized; retry storms continue |
| **Full jitter** | `random(0, min(cap, base·2ⁿ))` | Best load spreading; fewest total calls in practice | Some retries happen almost immediately; high per-client variance |
| **Equal jitter** | `t = min(cap, base·2ⁿ)` → `t/2 + random(0, t/2)` | Guarantees a minimum wait | Less spreading than full jitter; slightly more total work |
| **Decorrelated jitter** | `min(cap, random(base, prev_sleep·3))` | Good spread; adapts based on previous sleep | Stateful; bounds harder to reason about |

**Default recommendation:** full jitter with a cap. Use equal jitter when you need a guaranteed minimum wait (e.g. an upstream that is rate-limiting you).

### Implementation (Java)

```java
long backoffMs(int attempt) {
    long exp = Math.min(CAP_MS, BASE_MS * (1L << Math.min(attempt, 20)));
    return ThreadLocalRandom.current().nextLong(0, exp + 1);   // full jitter
}
```

Notes:
- `Math.min(attempt, 20)` prevents shift overflow.
- If the server sends `Retry-After`, treat it as the **floor** and add jitter on top.

---

## 4. Where Jitter Applies Beyond Retries

| Use case | How jitter is applied |
|---|---|
| **Cache TTLs** | `ttl = base ± random(10–20%)` so keys written together don't expire together |
| **Scheduled jobs / cron** | Random start offset within a window |
| **Leader election** | Raft uses randomized election timeouts (150–300ms) so candidates don't split votes forever |
| **Heartbeats / health checks** | Randomized interval prevents synchronized probe bursts |
| **Client polling / token refresh** | Refresh at a random point in 75–90% of lifetime, not exactly at expiry |
| **Metrics scraping** | Prometheus spreads scrape targets across the interval |
| **Mobile / IoT reconnects** | Essential after a regional outage when millions of devices reconnect at once |

---

## 5. Applied: Jitter in the Web Crawler

The crawler guide already uses jitter in one place ("Jitter TTL refresh" in the DNS cache). It belongs in several more:

```
                 ┌──────────────────────────────┐
                 │  Where jitter goes            │
                 └──────────────────────────────┘
  Fetcher 429/503 backoff ───► full jitter, Retry-After as floor
  Politeness delay ──────────► jitter UPWARD only (never below Crawl-delay)
  Adaptive recrawl T=1/λ ────► T · (1 ± 0.1·rand)
  Cold→hot promotion job ────► staggered start per shard
  robots.txt / DNS cache TTL ► ttl ± random(10–20%)
```

| Location | Without jitter | With jitter |
|---|---|---|
| **Fetcher backoff on 429/503** | Every fetcher that hit a struggling domain retries in lockstep | Retries spread out; honor `Retry-After` as the floor |
| **Politeness delay** | Fine on its own, but many domains align after a restart | `next_ok_ts = end_of_fetch + crawl_delay + random(0, ε)` — **only upward**. Never jitter below the robots.txt `Crawl-delay` |
| **Adaptive recrawl (`T = 1/λ`)** | Millions of pages crawled in one batch get similar intervals and come due together | `T · (1 ± 0.1·rand)` smooths frontier load |
| **Cold → hot frontier promotion** | Every shard scans Cassandra at the same moment | Staggered promotion per shard |
| **robots.txt cache TTL** | Deploy-time warmup means the whole cache expires at once → burst of robots.txt fetches | Jittered TTL spreads refetches |

---

## 6. Alternatives & Complements

Jitter is not always the best tool. A Staff-level answer compares it against these:

| Technique | When it beats or complements jitter |
|---|---|
| **Deterministic jitter** (`offset = hash(id) % window`) | Reproducible and debuggable; the same entity always gets the same slot. Good for scheduled jobs and per-domain recrawl offsets |
| **Request coalescing (singleflight)** | For cache stampedes: one request recomputes the key, others wait. Eliminates duplicate work instead of spreading it |
| **Probabilistic early expiration (XFetch)** | Recompute before expiry with probability rising as expiry approaches. Hot keys refresh smoothly with no stampede |
| **Stale-while-revalidate** | Serve stale data while one background refresh runs; users never see the miss |
| **Retry budgets** (e.g. retries ≤ 10% of requests) | Jitter spreads retries but doesn't cap them. Budgets bound total amplification |
| **Circuit breakers** | Stop retrying entirely when a dependency is clearly down. Jitter alone keeps poking a dead service |
| **Token bucket / rate limiting** | Hard ceiling on throughput. Jitter is statistical and gives no guarantee |

### Random vs Deterministic Jitter

| | Random jitter | Deterministic (hashed) jitter |
|---|---|---|
| Spread | Statistically even | Even if IDs are uniformly distributed |
| Reproducibility | Hard to reproduce timing bugs | Fully reproducible |
| Risk | Occasional unlucky clustering | Persistent hot slots if IDs are skewed |
| Best for | Retries, reconnects | Scheduled jobs, recrawl offsets, cron |

---

## 7. Failure Modes & Mitigations

| Failure | Symptom | Mitigation |
|---|---|---|
| Retry storm after recovery | Dependency recovers, then immediately falls over again | Full jitter + cap; retry budgets; circuit breaker |
| Retry amplification across layers | 4 layers × 3 retries = 3⁴ = **81 calls** for one user request | Retry at one layer only (usually the edge); budgets |
| Cache stampede on hot key | DB spike when a popular key expires | TTL jitter + singleflight + XFetch / stale-while-revalidate |
| Jitter violates a hard constraint | Requests sent faster than `Crawl-delay` / `Retry-After` | Jitter only upward from the floor |
| Jitter breaks staleness SLA | Data up to 20% staler than the stated bound | Include the jitter range in the documented staleness bound |
| Deploy-time phase lock | All pods' timers aligned for their lifetime | Randomize initial delay on startup |
| Hot slot in hashed jitter | One time slot overloaded due to skewed IDs | Salt the hash; combine with small random component |

---

## 8. Senior vs Staff Answer Differentiators

### Senior-level expectations

- Knows exponential backoff and that jitter should be added to it
- Can explain the thundering herd problem
- Applies TTL jitter to caches

### Staff-level expectations (additional)

| Topic | What Staff-level adds |
|---|---|
| Strategy choice | Compares full / equal / decorrelated jitter with tradeoffs |
| Limits of jitter | States explicitly that jitter spreads load but doesn't reduce it |
| Amplification | Raises cross-layer retry multiplication (3⁴ = 81) and retry budgets |
| Constraints | Jitters only upward from hard floors (`Crawl-delay`, `Retry-After`) |
| SLAs | Notes that TTL jitter widens the staleness bound |
| Alternatives | Brings in singleflight, XFetch, stale-while-revalidate, circuit breakers |
| Determinism | Discusses random vs hashed jitter and debuggability |
| Root cause | Asks whether the real fix is removing the shared trigger |

### The test question

> **"What happens 30 seconds after the dependency recovers?"**
> If the answer is "every client hits it at once," you need jitter.

---

## 9. Quick-Reference Cheatsheet

```
DEFINITION
──────────
Intentional randomness in timing → desynchronizes clients → spike becomes plateau
Spreads load, does NOT reduce it

BACKOFF FORMULAS
────────────────
No jitter:          min(cap, base·2ⁿ)
Full jitter:        random(0, min(cap, base·2ⁿ))           ← default
Equal jitter:       t/2 + random(0, t/2),  t = min(cap, base·2ⁿ)
Decorrelated:       min(cap, random(base, prev·3))

COMMON VALUES
─────────────
Cache TTL jitter:       ±10–20%
Token refresh:          random point in 75–90% of lifetime
Raft election timeout:  150–300ms randomized
Retry budget:           ≤ 10% of requests

RULES
─────
1. Always cap the backoff
2. Retry-After / Crawl-delay is a FLOOR — jitter only upward
3. Retry at one layer only
4. Pair with budgets + circuit breakers
5. For stampedes, prefer coalescing / early expiration over jitter alone
6. Document the staleness impact of TTL jitter

STAFF SIGNALS
─────────────
✓ Compare full / equal / decorrelated jitter
✓ "Jitter spreads load, doesn't reduce it"
✓ Retry amplification across layers (3⁴ = 81)
✓ Upward-only jitter for hard floors
✓ Random vs deterministic jitter tradeoff
✓ Singleflight, XFetch, stale-while-revalidate as alternatives
```

---

## 10. Related Guides

- **Web Crawler** — DNS cache TTL jitter, fetcher backoff, politeness delay, adaptive recrawl scheduling
- **Distributed Cache** — cache stampede, TTL jitter, singleflight, probabilistic early expiration
- **Rate Limiter** — token bucket as a hard ceiling vs jitter as statistical spreading
- **Distributed Consensus (Raft)** — randomized election timeouts

---

*Guide compiled for Senior & Staff-level system design interview preparation.*
*Topics: Jitter, Exponential Backoff, Thundering Herd, Retry Storms, Cache Stampede.*
