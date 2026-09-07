# Real-Time Telemetry & Threat Ingestion Pipeline — System Design Interview Guide
### Senior & Staff Engineer Level — Cloud Workload Security Lens (GuardDuty / CSPM-style)

---

## Table of Contents

1. [Problem Statement & Scope](#1-problem-statement--scope)
2. [Requirements](#2-requirements)
3. [Capacity Estimation](#3-capacity-estimation)
4. [High-Level Architecture](#4-high-level-architecture)
5. [Component 1: Data Sources & Collectors](#5-component-1-data-sources--collectors)
6. [Component 2: Ingestion Gateway & Schema Normalization](#6-component-2-ingestion-gateway--schema-normalization)
7. [Component 3: Multi-Tenant Streaming Bus](#7-component-3-multi-tenant-streaming-bus)
8. [Component 4: Rule-Based Detection Engine (Streaming CEP)](#8-component-4-rule-based-detection-engine-streaming-cep)
9. [Component 5: ML / Anomaly Detection Engine (UEBA)](#9-component-5-ml--anomaly-detection-engine-ueba)
10. [Component 6: Enrichment Service](#10-component-6-enrichment-service)
11. [Component 7: Correlation & Incident Stitching](#11-component-7-correlation--incident-stitching)
12. [Component 8: CSPM Posture Scanning (Config Drift)](#12-component-8-cspm-posture-scanning-config-drift)
13. [Component 9: Storage Layer](#13-component-9-storage-layer)
14. [Component 10: Alerting & Case Management](#14-component-10-alerting--case-management)
15. [CAP / PACELC Positioning](#15-cap--pacelc-positioning)
16. [Failure Modes & Mitigations](#16-failure-modes--mitigations)
17. [Scalability, Sharding & Multi-Tenancy](#17-scalability-sharding--multi-tenancy)
18. [Senior vs Staff Answer Differentiators](#18-senior-vs-staff-answer-differentiators)
19. [Interview Time Allocation](#19-interview-time-allocation)
20. [Quick-Reference Cheatsheet](#20-quick-reference-cheatsheet)

---

## 1. Problem Statement & Scope

A **real-time telemetry & threat ingestion pipeline** for cloud workload security ingests logs and runtime signals from thousands of customer cloud accounts — control-plane API activity, network flow logs, container/host runtime events, Kubernetes audit logs, and periodic resource configuration snapshots — normalizes them into a common schema, and runs both **streaming threat detection** and **batch posture/configuration scanning** to surface security findings. This is the system class underlying products like AWS GuardDuty, Microsoft Defender for Cloud, Wiz, Orca Security, Prisma Cloud, and Datadog Cloud SIEM (a **CNAPP** — Cloud-Native Application Protection Platform).

### What the interviewer is really testing

- Can you design a **multi-tenant, high-throughput streaming ingestion + detection pipeline**, not just a generic log pipeline?
- Do you understand that **streaming (event-based) detection** and **posture scanning (state-based)** are fundamentally different paradigms that need different infrastructure?
- Can you architect **two independent detection philosophies** — deterministic rules and statistical/ML anomaly detection — as first-class, separately scalable systems that converge downstream?
- Do you treat **tenant isolation and alert quality** as core architecture, not bolted-on concerns — this is what separates a security product from a generic analytics pipeline?
- Can you make explicit **CAP/PACELC** calls, recognizing that a security system has *more* CP-leaning components than a typical AP-mostly system?

---

## 2. Requirements

### Functional Requirements

| Requirement | Notes |
|---|---|
| Ingest heterogeneous telemetry | CloudTrail/API audit logs, VPC Flow Logs, DNS query logs, K8s audit logs, host/container runtime events (eBPF), periodic config snapshots |
| Normalize into a common schema | Uniform detection logic across sources and cloud providers |
| Real-time streaming detection | Rule-based (deterministic) AND ML/anomaly-based, as independent paths |
| Batch/periodic posture scanning | Detect misconfigurations and config drift (CSPM) — a different paradigm than event detection |
| Correlate findings into incidents | Kill-chain/attack-graph stitching; avoid presenting 7 alerts for 1 attacker action |
| Enrich findings | Threat intel (IP/domain/hash reputation) + asset/identity context |
| Suppress & deduplicate | Analyst-authored suppression must be honored before delivery |
| Deliver to SIEM/case management | Severity-scored findings pushed to customer tooling |
| Retroactive detection | Replay new rules/IOCs against historical raw telemetry |

### Non-Functional Requirements

| Requirement | Target |
|---|---|
| Scale | ~10,000+ monitored cloud accounts, tens of billions of events/day |
| Multi-tenancy | Strict data isolation — cross-tenant leakage is a catastrophic failure, not a bug |
| Detection latency | Sub-minute for high-severity streaming findings; minutes-to-hours acceptable for posture scans |
| Durability | Raw telemetry is the forensic record — must never be silently dropped |
| Alert quality | False-positive rate is a first-class SLO — alert fatigue causes real analysts to ignore real threats |
| Retroactivity | New detections must be replayable against recent historical data without re-ingesting |
| Availability bias | Never drop raw ingestion under load; degrade detection compute gracefully instead |

---

## 3. Capacity Estimation

*(Numbers below are illustrative — the reasoning process matters more than the specific figures.)*

```
Target: 10,000 monitored cloud accounts (AWS/Azure/GCP mix)
Blended average: ~5M combined events/account/day
  (small dev accounts: thousands/day; large prod accounts: 100M+/day)

Total volume:
  10,000 × 5,000,000 ≈ 50 billion events/day

Throughput:
  50,000,000,000 / 86,400 ≈ 578,700 events/second (avg)
  Peak (business-hours + incident-driven bursts, ~3x avg): ~1.7M events/sec

Bandwidth (avg 800 bytes/event, normalized + compressed):
  578,700 × 800B ≈ 463 MB/s avg  →  peak ≈ 1.4 GB/s

Kafka partitions (target ~20MB/s/partition sustained throughput):
  1.4 GB/s / 20MB/s ≈ 70 partitions minimum
  → provision 256 partitions for headroom + per-tenant isolation

Rule engine (Flink) task slots:
  Assume 1 slot evaluates ~10K events/sec against the active rule set
  578,700 / 10,000 ≈ 58 slots avg → provision ~200 slots for peak + backpressure headroom

UEBA feature store sizing:
  ~50M active (identity, resource) pairs under observation
  ~2KB/entry rolling stats (window counts, EWMA, categorical sketches)
  ≈ 100 GB working set per detection job — fits a sharded Redis/RocksDB-backed state store

Raw event lake:
  800 bytes/event × 50B events/day ≈ 40 TB/day compressed raw
  90-day hot retention → ~3.6 PB rolling (object store, tiered after 90 days)

Threat-intel enrichment load:
  Peak 1.7M events/sec × ~2 IOC lookups/event (IP + domain) ≈ 3.4M lookups/sec
  → Bloom-filter fast-reject in front of a hot IOC cache (~500M indicators)
```

**Key insight:** unlike a generic log pipeline, the dominant *design* cost here isn't raw throughput — it's that three very different consumption patterns (low-latency streaming rules, stateful ML feature aggregation, and periodic batch posture scans) must all read from the same normalized stream without one starving the others.

---

## 4. High-Level Architecture

```
   Customer Cloud Accounts (AWS / Azure / GCP) — 10,000+ tenants
        │
        ├── CloudTrail / Activity Logs ────────┐
        ├── VPC Flow Logs / DNS query logs ─────┤
        ├── Kubernetes audit logs ───────────────┼──▶  Collectors
        ├── Host/container runtime (eBPF) ───────┘      (per-source adapters)
        └── Periodic config snapshot polling  ───────────────┐  (separate path, §12)
                          │                                   │
                          ▼                                   │
              ┌─────────────────────────┐                     │
              │   Ingestion Gateway      │                     │
              │   Parse + Normalize      │                     │
              │   → OCSF common schema   │                     │
              └────────────┬─────────────┘                     │
                            │ (Kafka, partitioned by tenant_id) │
                            ▼                                   │
              ┌───────────────────────────────────┐             │
              │      Multi-Tenant Streaming Bus     │             │
              └──────┬────────────────────┬─────────┘             │
                     │                    │                       │
          ┌──────────▼─────────┐  ┌───────▼─────────────┐         │
          │  Rule-Based Engine  │  │  ML/Anomaly Engine   │         │
          │  (Flink CEP,        │  │  (feature store +    │         │
          │   Sigma rules)      │  │   2-stage scoring)   │         │
          └──────────┬─────────┘  └───────┬─────────────┘         │
                     │                    │                       │
                     └─────────┬──────────┘                       │
                               ▼                                  │
                    ┌────────────────────┐                        │
                    │  Enrichment Service │                        │
                    │  threat intel +     │◀───────────────────────┘
                    │  asset/identity ctx │      (asset inventory feed)
                    └──────────┬──────────┘
                               ▼
                    ┌────────────────────┐        ┌─────────────────────┐
                    │  Correlation &      │◀───────│  CSPM Posture Engine │
                    │  Incident Stitching │        │  (policy-as-code vs. │
                    └──────────┬──────────┘        │   config snapshots)  │
                               │                    └─────────────────────┘
              ┌────────────────┴─────────────────┐
              ▼                                   ▼
   ┌─────────────────────┐             ┌───────────────────────┐
   │   Storage Layer       │             │  Alerting & Case       │
   │  Raw lake (S3)         │             │  Management            │
   │  Hot index (OpenSearch)│             │  (SIEM push, severity,│
   │  Graph store (Neptune) │             │   suppression rules)  │
   └─────────────────────┘             └───────────────────────┘
```

**Two independent detection paths** (rule-based and ML) consume the same normalized stream in parallel and converge only at the enrichment/correlation layer — this split, and the reasoning for keeping them architecturally separate rather than one monolithic "detection service," is the single most important thing to draw and narrate early.

---

## 5. Component 1: Data Sources & Collectors

### Source Types & Native Characteristics

| Source | Delivery model | Native latency | Notes |
|---|---|---|---|
| CloudTrail / Azure Activity Log / GCP Audit Log | Batched file delivery to object storage + event notification | 5–15 min | API-call record of everything done in the account |
| VPC Flow Logs / DNS query logs | Batched delivery, similar pull model | 1–10 min | Network-layer visibility |
| Kubernetes audit logs | Webhook push from API server, or node-level log tail | Seconds | Cluster control-plane activity |
| Host/container runtime (eBPF sensor) | Persistent agent, gRPC stream to regional collector | Sub-second | Process exec, network connect, file access — the only *true* real-time source |
| Config snapshots | Scheduled poller hitting cloud provider describe/list APIs | Minutes–hours (poll interval) | Agentless CSPM model (cross-account IAM role assumption per tenant) |

### Staff-Level Gotcha: Reconciling Native Latency, Not Ignoring It

A Senior-level answer treats ingestion as uniformly "real-time." A **Staff-level answer explicitly reconciles the latency mismatch**: CloudTrail is *inherently* 5–15 minutes behind the actual API call no matter how fast your pipeline is downstream — you cannot promise sub-minute detection for a CloudTrail-only finding. Only eBPF runtime telemetry supports true sub-second detection. The system's detection-latency SLO must be **stated per source**, not as one blanket number, and finding severity/response playbooks should account for which source(s) contributed.

### Collector Responsibilities

- Per-source parsing adapters (isolate format churn to one adapter, not the whole pipeline)
- Ack-based pull with backpressure (SQS visibility timeout / equivalent) so a slow downstream never causes data loss, only delay
- Regional collection honoring data residency (EU tenant data collected and processed in-region for GDPR)
- Per-tenant credential/role isolation — collectors assume a scoped cross-account role per tenant, never a shared over-privileged credential

---

## 6. Component 2: Ingestion Gateway & Schema Normalization

### Why Normalize to a Common Schema

Without normalization, every detection rule would need N × M variants (N sources × M cloud providers). The industry-standard answer here is **OCSF (Open Cybersecurity Schema Framework)** — AWS Security Lake and most modern CNAPP vendors converge on it. Every raw record is mapped to a canonical event with:

```
Normalized event = {
  category:      "API Activity" | "Network Activity" | "Process Activity" | ...
  class:         specific event class within category
  time:          event_time (from source, NOT ingestion time)
  actor:         { identity, credential_type, session_context }
  resource:      { id, type, cloud_provider, account_id, region }
  tenant_id:     stamped here — drives all downstream partitioning
  severity_hint: source-provided severity if any
  raw_ref:       pointer back to raw archived record for drill-down
}
```

### Handling Schema Drift

Cloud providers change log formats without warning (new CloudTrail fields, renamed VPC Flow Log columns). A schema registry enforces backward-compatible evolution. Malformed or unparseable records are routed to a **dead-letter topic**, never blocking the main stream — the same poison-pill isolation principle as any high-throughput ingestion system: one malformed record must never stall valid ones behind it.

### Tenant Tagging Happens Here — Not Later

`tenant_id` is stamped at the earliest possible point and treated as a **mandatory, schema-enforced field** for every downstream hop. This is the single control point that makes tenant isolation auditable later — see the cross-tenant leak failure mode in §16.

---

## 7. Component 3: Multi-Tenant Streaming Bus

### Partition by Tenant, Not by Hash

The bus (Kafka or Kinesis) is partitioned by `tenant_id`, not by an even hash of the event key. This mirrors a familiar lesson from any high-throughput streaming design: **locality enables local state**. Here, tenant-local partitioning enables:

- Per-tenant backpressure and quota enforcement (one noisy tenant can't starve others)
- Per-tenant retroactive replay (re-run a new rule for one customer without touching the rest)
- A clean boundary for the tenant-isolation guarantee itself

### The Large-Tenant Problem

A single enterprise tenant can dwarf thousands of small tenants combined in event volume — the direct analog of a hot-key/hot-partition problem. **Fix:** once a tenant's volume crosses a threshold, sub-partition it by `(tenant_id, source_type)` or `(tenant_id, shard_suffix)`, giving it dedicated partitions and downstream compute rather than sharing infrastructure with hundreds of small tenants (a **noisy-neighbor isolation** concern that's specific to a multi-tenant security product — a small tenant's detection latency SLO must not degrade because a large tenant is having an incident).

### Topic Layout

- `normalized-events` — single topic across all sources, tagged by category; enables cross-source correlation (an IAM event correlated with a network event) without cross-topic joins
- `raw-{source}` — per-source raw archival topics, immediately sinked to the object store for compliance/forensic retention
- Hot retention: 7 days on `normalized-events` (supports near-real-time rule replay); raw archival retained per the compliance policy (often years)

---

## 8. Component 4: Rule-Based Detection Engine (Streaming CEP)

### Sigma Rules as the Detection DSL

**Sigma** is the industry-standard, YAML-based detection rule format, portable across SIEM backends. Rules compile into a streaming job graph (Flink, or Kafka Streams for simpler stateless rules):

```
Example rule semantics (Sigma-style):
  detection:
    category: API Activity
    action: AssumeRole
    condition: source_ip NOT IN known_ip_ranges(identity)
  correlation:
    group_by: [actor.identity]
    window: 5m
    condition: distinct(geo(source_ip)) >= 2   # "impossible travel"
```

Multi-event correlation (like impossible travel) requires **keyed, windowed state** — Flink's keyed state + CEP pattern-matching library is the natural fit, holding a per-principal rolling window of recent source geolocations with TTL eviction.

### Retroactive Replay — A Named Requirement, Not an Afterthought

When a new rule ships (e.g., a newly published MITRE ATT&CK technique), it must be run against recent historical raw data to surface pre-existing matches — waiting for new events to trickle in is unacceptable for a fast-moving threat. This is implemented as a **bounded batch job using the identical compiled rule logic** against the raw archive (§13), executed on a separate compute pool so it never competes with live streaming detection for resources (see the replay-overload failure mode in §16).

### Pros / Cons (state explicitly — interviewers want the tradeoff, not just the mechanism)

| | Pros | Cons |
|---|---|---|
| Rule-based | Interpretable, deterministic, auditable (analysts can see *why* it fired), fast to author against newly published TTPs, zero training data | Only catches *known* patterns; trivially evaded by minor technique variation; rule sprawl becomes a maintenance burden |

---

## 9. Component 5: ML / Anomaly Detection Engine (UEBA)

**UEBA (User & Entity Behavior Analytics)** builds a rolling behavioral baseline per identity/resource and flags statistically unusual deviation — this is the path that catches *unknown* threats the rule engine was never told to look for.

### Two-Stage Architecture (the explicit split the interviewer wants to see)

```
Stage 1 — Streaming, cheap, high recall / low precision:
  Per-entity rolling stats (mean, stddev, EWMA, categorical
  novelty via count-min sketch / HyperLogLog) computed inline
  in the streaming layer. Flags candidates on every event.

Stage 2 — Async, more expensive, higher precision:
  Candidates from Stage 1 are scored by a heavier model
  (e.g., isolation forest / gradient-boosted classifier)
  trained offline on labeled feature vectors. Runs on the
  much smaller candidate set within seconds-to-minutes.
```

Favor simple, explainable per-entity statistics for the fast layer over a single black-box global model — it's operationally tractable at this scale and gives analysts a legible reason ("this identity's API call volume is 8 standard deviations above its 30-day baseline") rather than an opaque score.

### Two Staff-Level Gotchas

**Model/baseline drift:** a tenant's "normal" legitimately shifts (org restructuring, new project rollout). Without an EWMA-style decay on the baseline, false-positive rate silently climbs after any organizational change — this must be designed in from the start, not patched in after analysts complain.

**Cold-start problem:** a brand-new tenant or a newly created IAM role has no baseline. Anomaly detection is *unusable* until a sufficient observation window has passed (typically 7–14 days). The system must **explicitly fall back to rule-based-only detection** during this window — a silent gap in coverage is far worse than an acknowledged, temporary one.

### Cardinality Explosion (the domain's "spider trap")

An attacker fanning out across thousands of ephemeral, short-lived IAM roles can blow up the per-entity feature store trying to build individual baselines for each. **Mitigation:** cap cardinality and roll up ephemeral entities into a parent-entity baseline (group by role *template*, not by individual assumed-role session) — the direct domain analog of a crawler's spider-trap detection.

---

## 10. Component 6: Enrichment Service

### Threat Intelligence Lookups

IP/domain/file-hash reputation checked against curated and commercial feeds. Use the same performance pattern as any high-throughput set-membership check: a **Bloom filter for fast reject**, backed by a full KV lookup (Redis) for the smaller set of likely hits. This keeps the ~3.4M lookups/sec peak load (§3) off a slow full-lookup path for the vast majority of clean IPs/domains.

### Asset & Identity Context

Raw resource IDs are enriched with human-meaningful context — team ownership, production-vs-dev classification, human-vs-machine identity, resource tags — pulled from a continuously refreshed **asset inventory**, itself fed by the CSPM config snapshot poller (§12). This is what turns "`AssumeRole` from `1.2.3.4`" into "production billing-service role assumed from an IP outside documented CI/CD egress ranges" — the enrichment is what makes a finding *actionable*.

### Enrichment Must Never Block the Hot Path

Async, cache-first, best-effort. If a threat-intel call times out, emit the finding **unenriched with a fallback flag** rather than blocking or dropping it — a lower-context finding beats a lost finding. (A circuit breaker in front of external threat-intel calls is the standard pattern here — fail fast to "unenriched" rather than let a slow external dependency back up the whole detection pipeline.)

---

## 11. Component 7: Correlation & Incident Stitching

### The Alert-Fatigue Problem

A single attacker action often trips multiple independent detections — a compromised credential doing reconnaissance might fire 5 separate rule matches plus 2 anomaly flags over 10 minutes. Presenting these as **7 separate alerts** is the single biggest driver of analyst alert fatigue, and alert fatigue is what causes real, dangerous incidents to be ignored.

### Graph-Based Grouping

Findings sharing common entities (same principal, same source IP, same resource) within a rolling time window are grouped into a single incident. Model findings and entities as nodes/edges; a **connected-components** pass over the rolling-window graph produces the grouping.

### MITRE ATT&CK Stage Tagging

Each finding is tagged with an attack-chain stage (Initial Access, Discovery, Exfiltration, etc.). An incident spanning **multiple stages** for the same actor is scored far more severely than isolated single-stage findings — this surfaces *progression*, not just volume, and is a strong differentiator between a system that merely lists alerts and one that tells a story.

### Idempotent Incident Dedup

The same underlying incident must not be recreated when a streaming job reprocesses a window after a restart. Use a **deterministic incident key** (hash of the grouped entity set + time bucket) with upsert semantics — never append-only insert — so reprocessing merges into the existing incident instead of duplicating it.

---

## 12. Component 8: CSPM Posture Scanning (Config Drift)

### A Fundamentally Different Paradigm — State-Based, Not Event-Based

This is the point most Senior-level candidates flatten into "just another detector." It isn't: posture scanning asks *"is the current configuration of this resource compliant?"* — independent of any recent event. It needs its own scheduler and its own diffing logic, not the streaming detection infrastructure.

```
Config Snapshot Poller (§5) ──▶ Policy Engine ──▶ Drift Diff ──▶ Correlation Layer (§11)
  (scheduled, cross-account       (policy-as-code,    (compare to
   IAM role assumption)            e.g. OPA/Rego)       last-known-good)
```

- **Poller:** incremental snapshots triggered by CloudTrail config-change events for fast-changing resources (security groups, IAM policies), plus a full periodic sweep every N hours to catch anything missed.
- **Policy engine:** evaluates snapshots against policy-as-code rules — e.g., "flag any S3 bucket with public-read ACL," "flag any security group allowing `0.0.0.0/0` on port 22."
- **Resource relationship graph:** a posture finding is far more actionable with blast-radius context. "This public S3 bucket contains customer PII and is reachable from the internet-facing web tier" requires a graph connecting network topology, the IAM policy graph (who/what can reach this resource), and data-classification tags — stored in a graph database (Neptune/Neo4j), overwritten each snapshot cycle rather than event-logged.
- **Drift diffing (avoiding the re-alert trap):** compare the current snapshot to the last-known-good one and flag only the *delta* as a new finding. Without this, every pre-existing misconfiguration re-fires on every poll cycle — a self-inflicted alert-fatigue trap distinct from, but just as damaging as, the correlation problem in §11.

---

## 13. Component 9: Storage Layer

| Layer | What | System | Rationale |
|---|---|---|---|
| Raw event lake | Full normalized + raw records | S3/GCS, partitioned by `tenant_id/date/source_type` | Forensic record; enables retroactive rule replay; append-only, WORM-style retention for regulated tenants |
| Hot findings/incidents index | Recent findings for analyst search | OpenSearch/Elasticsearch | Full-text + faceted search, dashboarding; tiered to cold storage after N days |
| Resource relationship graph | Current-state network/IAM/data-classification graph | Neptune / Neo4j | Blast-radius queries for posture and incident context; overwritten per snapshot cycle, not event-logged |
| UEBA feature store | Rolling per-entity behavioral baselines | Redis / DynamoDB, TTL'd | Treated as a rebuildable cache, not source of truth — reconstructable from raw event replay if lost |
| Suppression & isolation config | Analyst suppression rules, tenant ACLs | Strongly-consistent store (e.g., a CP-configured KV/DB) | See §15 — these are the named CP exceptions |

### Retention Policy

```
Raw event lake:        90 days hot → cold/archive tier (compliance-driven, often years)
Findings/incidents:    Rolling 1–2 years (analyst investigation window)
Resource graph:        Current-state only; historical snapshots retained separately for drift audit
UEBA feature store:    Rolling window only (7–90 days depending on baseline window) — never a system of record
```

---

## 14. Component 10: Alerting & Case Management

### Severity Scoring

Combines: rule/ML confidence, MITRE ATT&CK stage span (§11), asset criticality (from the asset inventory, §10), and blast radius (from the resource graph, §12). A single formula fed by four independently-maintained inputs — not a single detector's confidence score — is what makes severity trustworthy enough for analysts to triage by.

### Delivery

Push to customer SIEM (API/webhook), ticketing (Jira/ServiceNow connector), or a native case-management UI.

### Suppression — Consulted *Before* Delivery, Not After

Analyst-authored suppression ("mute this finding type for this resource — accepted risk") must be checked before a finding is delivered, not filtered out after the fact. Suppression state must be **strongly consistent** (§15) — a stale suppression rule that fails to apply re-alerts on an already-accepted risk and erodes analyst trust in the product; this is treated as a correctness property, not a performance nice-to-have.

---

## 15. CAP / PACELC Positioning

Most of this system is **AP/EL** — but a security product genuinely has more named **CP** exceptions than a typical AP-leaning system, and calling that out explicitly (with *why*) is a strong Staff signal.

| Component | CAP / PACELC | Reasoning |
|---|---|---|
| Ingestion gateway / raw bus | **AP / EL** | Never drop raw telemetry; availability over consistency, eventual consistency downstream is fine |
| Multi-tenant streaming bus | **AP / EL** | Tunable per-topic acks; durability via replication, not synchronous consistency |
| Rule-based & ML detection compute | **AP / EL** | Best-effort, eventually-consistent detection; a few seconds of lag under load is acceptable |
| Raw event lake (S3) | **AP / EL** | Append-only, writes never conflict |
| CSPM config snapshot/graph | **AP / EL** | Staleness bounded by poll interval; acceptable |
| Threat-intel cache | **AP / EL** | Stale IOC entry for a short TTL window is an acceptable tradeoff for availability |
| UEBA feature store | **AP / EL** | Rebuildable from raw replay; not a source of truth |
| **Correlation / incident-dedup store** | **CP** | Must avoid duplicate or split incidents on reprocessing; under partition, pause creation rather than risk incorrect merges |
| **Suppression rule store** | **CP** | Stale suppression state re-alerting a muted, accepted-risk finding erodes trust; brief delay for a consistent check is preferable |
| **Tenant isolation / access-control config** | **CP** | The most important exception: a stale ACL permitting cross-tenant visibility is a severe security breach, not a tolerable staleness window |
| Coordination (job/shard assignment — ZooKeeper/etcd under Flink/Kafka) | **CP** | Shard/leader decisions must be consistent; availability sacrificed during partition |

**Staff-level framing:** in most distributed systems, CP is the narrow exception carved out of an otherwise AP/EL design. In a security product, the exceptions cluster specifically around **isolation boundaries and analyst-facing correctness guarantees** (who can see what, what's been muted, what's been merged) — the detection *compute* itself can stay loose, but the *control* surfaces around it cannot.

---

## 16. Failure Modes & Mitigations

| Failure | Symptom | Detection | Mitigation |
|---|---|---|---|
| Collector outage for one region/source | Silent gap in raw telemetry | Per-source freshness/heartbeat SLO monitoring | Dead-man's-switch alerting on missing freshness; backfill via re-pull once source recovers |
| Schema drift from cloud provider | Parsing breaks or silently mis-maps fields | Schema registry validation failures spike | Dead-letter queue for unparseable records; versioned schema with backward-compatibility checks |
| Noisy tenant / event storm | One tenant's misconfigured pipeline floods a shared partition, delaying others | Per-tenant lag/throughput metrics | Per-tenant quota + backpressure; sub-partition oversized tenants (§7) |
| IOC feed poisoning | A bad threat-intel entry causes mass false positives across all tenants at once | Sudden spike in finding rate correlated with a feed update | Canary/staged rollout of feed updates; per-feed confidence scoring; automatic rollback on FP-rate spike |
| Clock skew across regions/collectors | Incorrect impossible-travel or time-window correlation results | Correlation false-positive/negative rate anomalies | Event-time (not processing-time) windows; NTP-synced sources; watermarking for late-arriving events |
| UEBA cardinality explosion | Attacker fans out across ephemeral identities to evade per-entity baselining | Feature store memory/OOM alerts | Cardinality caps; roll up ephemeral entities to a parent-entity baseline |
| Cross-tenant data leak | Partition/cache-key/index-alias misrouting exposes one tenant's data to another | Automated cross-tenant access canaries (synthetic tenant probes) | `tenant_id` enforced by schema at every hop; per-tenant encryption keys; isolation config is CP (§15) |
| Correlation engine reprocesses after restart | Duplicate incidents flood analysts | Incident-creation rate spike correlated with job restart | Idempotent incident keys with upsert semantics; exactly-once via checkpointing/transactional writes |
| Retroactive rule replay overload | Replaying a new rule against 90 days of history for all tenants saturates compute | Replay job resource contention alerts | Throttled, prioritized replay on a separate compute pool from live detection |
| Cloud API throttling on the config poller | Posture data goes stale | Increasing snapshot age per resource | Exponential backoff; per-account API budget; prioritize fast-changing resource types over slow-changing ones |

---

## 17. Scalability, Sharding & Multi-Tenancy

### Tenant-Based Sharding Is the Primary Axis

```
shard_id = hash(tenant_id) % num_shards

Each shard owns:
  • Its Kafka partition set
  • Its Flink task slots (rule engine + UEBA feature computation)
  • Its slice of the feature store

Large-tenant special case:
  Tenants above a volume threshold get dedicated Kafka
  partitions and dedicated compute — never share infra
  with hundreds of small tenants (noisy-neighbor isolation)
```

### Horizontal Scaling Plan

| Component | Scaling Strategy |
|---|---|
| Ingestion gateway | Stateless; add nodes behind the collector queues |
| Streaming bus | Add Kafka partitions/brokers; sub-partition oversized tenants |
| Rule engine (Flink) | Add task managers; rules are stateless per-key, scale by keyed parallelism |
| UEBA engine | Shard feature store by entity key; add Stage-2 scoring workers independently of Stage-1 |
| Enrichment service | Stateless; scale with cache hit-rate monitoring |
| Correlation engine | Shard by entity-graph partition; windowed state, similar scaling profile to the rule engine |
| CSPM poller | Scale by account count; bounded by per-account cloud API rate limits, not compute |
| Raw event lake | Managed object store; scales automatically |
| Resource graph | Add graph DB replicas for read scaling |

### Data Residency Adds a Second Sharding Constraint

A purely hash-based shard assignment optimizes for load balance alone. Here, **EU tenant data must be collected, processed, and stored in an EU region** (GDPR) — so the sharding key must satisfy *both* load-balance and regional-affinity constraints simultaneously, which can be in tension (an EU region with fewer large tenants may be under-utilized relative to a US region with several giant ones). This is a domain-specific nuance worth raising proactively: pure consistent hashing isn't sufficient on its own; shard assignment needs a region-aware constraint layer on top.

---

## 18. Senior vs Staff Answer Differentiators

### Senior-level expectations

- Correctly identify the major components (ingestion, normalization, detection, storage, alerting)
- Mention rules and ML as detection methods
- Understand basic multi-tenancy via an `account_id`/`tenant_id` field
- Handle obvious failure modes (dead source, malformed record)
- Provide basic capacity math

### Staff-level expectations (additional)

| Topic | What Staff-level adds |
|---|---|
| Detection architecture | Explicit two-path (rule + ML) design with distinct infra, latency, and precision/recall tradeoffs, converging only at correlation |
| Multi-tenancy | Tenant-based sharding, per-tenant quotas, dedicated shards for large tenants, cross-tenant leak as a named failure mode with canaries |
| CAP/PACELC | Explicit CP carve-outs specifically for isolation/access-control and incident/suppression state — and *why* security systems skew more CP than typical AP-mostly systems |
| Alert quality | False-positive rate treated as a first-class SLO; correlation/dedup/suppression designed as core architecture, not bolted on |
| Retroactive detection | Explicit raw-event-lake replay capability, throttled on separate compute from live detection |
| Posture vs streaming | Recognizes CSPM as a fundamentally different state-based paradigm requiring its own scheduler and drift-diff logic |
| Cold-start & drift | Explicit rule-only fallback during UEBA baseline cold-start; EWMA decay for legitimate behavioral shift |
| Cross-source latency | Reconciles CloudTrail's 5–15 min native lag against eBPF's sub-second latency with per-source SLOs, not one blanket number |
| Data residency | Sharding key must satisfy both load-balance and regional-affinity constraints, not hash-only |

### The single most important Staff differentiator

**Treating alert quality and tenant isolation as architecture, not afterthoughts.** Any engineer can wire up ingestion → detection → alert. The Staff-level signal is designing correlation, suppression, and isolation as first-class systems from the start — because in a security product, a false-positive flood and a cross-tenant leak are not edge cases to patch later; they're the failure modes that end the product.

---

## 19. Interview Time Allocation

For a 45-minute system design session:

| Phase | Time | Focus |
|---|---|---|
| Requirements & scope | 5 min | Confirm sources, multi-tenancy, streaming vs. posture distinction |
| Capacity estimation | 5 min | Events/sec, bandwidth, partitions, feature-store sizing |
| High-level architecture | 5 min | Draw all components; name the two detection paths explicitly |
| Ingestion & normalization | 5 min | OCSF, schema drift handling, tenant tagging |
| Detection engines (deep dive) | 12 min | Rule-based CEP + UEBA two-stage design; cold-start; retroactive replay |
| Correlation & alert quality | 8 min | Incident stitching, MITRE stage tagging, suppression, dedup idempotency |
| CSPM posture scanning | 3 min | State-based paradigm, drift diffing |
| CAP positioning & failure modes | 5 min | Named CP exceptions; cross-tenant leak, noisy tenant, IOC poisoning |
| Multi-tenancy & sharding (if time) | 2 min | Tenant-based sharding, data residency constraint |

**What to cut if short on time:** posture-scanning depth and interview-time on capacity math beyond the headline numbers. **Never cut:** the rule/ML architecture split, correlation/alert-quality design, or the named CAP exceptions around isolation.

---

## 20. Quick-Reference Cheatsheet

```
KEY ARCHITECTURAL SPLIT
────────────────────────
Two independent detection paths, same normalized stream:
  Rule-based (Flink CEP, Sigma)      — deterministic, interpretable, known patterns
  ML/Anomaly (2-stage UEBA)          — statistical, catches unknown patterns
  → converge only at Enrichment/Correlation

KEY ALGORITHMS
──────────────
Detection rules:     Sigma (YAML DSL)  →  compiled to Flink CEP job graph
UEBA Stage 1:        streaming z-score/EWMA + count-min sketch, high recall
UEBA Stage 2:        offline-trained model (isolation forest / GBM) on candidates
Incident grouping:   connected-components over a rolling entity/finding graph
IOC matching:        Bloom filter fast-reject + Redis full lookup

KEY NUMBERS (illustrative)
───────────────────────────
10,000 accounts × 5M events/day avg  →  ~578K events/sec avg, ~1.7M peak
1.4 GB/s peak bandwidth  →  256 Kafka partitions provisioned
~100 GB UEBA feature-store working set (50M active entity pairs)
40 TB/day raw compressed  →  ~3.6 PB rolling 90-day hot retention

KEY SYSTEMS
───────────
Schema:            OCSF (Open Cybersecurity Schema Framework)
Streaming bus:      Kafka, partitioned by tenant_id
Detection compute:  Flink (stateful CEP + windowed feature aggregation)
Raw lake:           S3/GCS, partitioned by tenant/date/source
Hot search:         OpenSearch/Elasticsearch
Resource graph:      Neptune/Neo4j (current-state, not event-logged)
Feature store:      Redis/DynamoDB (rolling, rebuildable cache)
Policy engine:      OPA/Rego or equivalent policy-as-code

CAP DECISIONS
─────────────
AP/EL:  ingestion, streaming bus, detection compute, raw lake, config snapshots,
        threat-intel cache, feature store
CP:     correlation/incident-dedup store, suppression rule store,
        tenant isolation/access-control config, job/shard coordination

DOMAIN-SPECIFIC CORRECTNESS TRAPS
───────────────────────────────────
✓ Event storm from one noisy tenant (spider-trap analog)
✓ IOC feed poisoning → mass false positives
✓ Clock skew → incorrect impossible-travel detection
✓ UEBA cardinality explosion via ephemeral identity fan-out
✓ Re-alerting on static posture findings without drift diffing
✓ Cross-tenant leak via partition/cache/index misrouting

STAFF-LEVEL SIGNALS TO HIT
───────────────────────────
✓ Explicit rule vs. ML architecture split with stated tradeoffs
✓ Posture scanning framed as a state-based, not event-based, paradigm
✓ Alert quality (correlation, suppression, dedup) as core architecture
✓ Named CP exceptions for isolation/suppression/incident state, with reasoning
✓ Per-source detection-latency SLOs, not one blanket "real-time" claim
✓ Cold-start fallback and baseline drift decay for UEBA
✓ Retroactive rule/IOC replay on a separate compute pool from live detection
✓ Data residency as a second constraint on top of hash-based sharding
```

---

## Related Guides

- **Apache Kafka** — streaming bus internals, `acks`/`min.insync.replicas` durability contract referenced in §7 and §15
- **CAP / PACELC Theorem** — general framework underlying §15's per-component positioning
- **Sharding Strategies** — general tenant/hash sharding patterns underlying §17
- **Distributed Key-Value Store (Dynamo-style)** — replication/consistency model applicable to the feature store and suppression/isolation config stores
- **Circuit Breaker Pattern** — applied to threat-intel enrichment calls in §10 to keep a slow external dependency from backing up the pipeline
- **Leader Election in Distributed Systems** — underlies Flink JobManager/coordinator behavior referenced in §15's coordination CP exception
- **Web Crawler** — structural parallels: domain-based partitioning ↔ tenant-based partitioning; Bloom-filter dedup ↔ Bloom-filter IOC matching; malformed-URL dead-letter handling ↔ schema-drift dead-letter handling; spider traps ↔ event storms/cardinality explosion
- **Endpoint Telemetry Pipeline** — covers the host-agent collection side (eBPF sensor design, agent fleet management) in more depth than §5 of this guide

---

*Guide compiled for Senior & Staff-level system design interview preparation.*
*Topics: Cloud Workload Security, CNAPP, CSPM, UEBA, Multi-Tenant Streaming, Threat Detection, CAP Theorem.*
