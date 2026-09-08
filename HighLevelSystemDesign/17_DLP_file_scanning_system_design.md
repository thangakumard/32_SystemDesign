# Distributed Data Loss Prevention (DLP) & File Scanning System — System Design Interview Guide
### Senior & Staff Engineer Level

---

## Table of Contents

1. [Problem Statement & Scope](#1-problem-statement--scope)
2. [Requirements](#2-requirements)
3. [Capacity Estimation](#3-capacity-estimation)
4. [High-Level Architecture](#4-high-level-architecture)
5. [Anchor Decision: Inline Enforcement vs Out-of-Band Detection](#5-anchor-decision-inline-enforcement-vs-out-of-band-detection)
6. [Component 1: Interception Points](#6-component-1-interception-points)
7. [Component 2: Content Extraction & Normalization Pipeline](#7-component-2-content-extraction--normalization-pipeline)
8. [Component 3: Classification Engine](#8-component-3-classification-engine)
9. [Component 4: Policy Engine & Risk Scoring](#9-component-4-policy-engine--risk-scoring)
10. [Component 5: Malware/AV Scanning Integration](#10-component-5-malwareav-scanning-integration)
11. [Component 6: Decision & Enforcement Actions](#11-component-6-decision--enforcement-actions)
12. [Component 7: Incident & Case Management / SIEM Integration](#12-component-7-incident--case-management--siem-integration)
13. [Component 8: Data Stores](#13-component-8-data-stores)
14. [Component 9: Feedback Loop & Model Tuning](#14-component-9-feedback-loop--model-tuning)
15. [CAP / PACELC Positioning](#15-cap--pacelc-positioning)
16. [Failure Modes & Mitigations](#16-failure-modes--mitigations)
17. [Scalability & Sharding](#17-scalability--sharding)
18. [Data Residency & Jurisdictional Constraints](#18-data-residency--jurisdictional-constraints)
19. [Senior vs Staff Answer Differentiators](#19-senior-vs-staff-answer-differentiators)
20. [Interview Time Allocation](#20-interview-time-allocation)
21. [Quick-Reference Cheatsheet](#21-quick-reference-cheatsheet)
22. [Related Guides](#22-related-guides)

---

## 1. Problem Statement & Scope

A **Data Loss Prevention (DLP) & File Scanning System** inspects content **in motion** (email, network egress, cloud app traffic), **at rest** (SaaS repositories, file shares, object storage), and **in use** (clipboard, USB, print, screen capture) to detect sensitive data — PII, PCI, PHI, source code, credentials, intellectual property — leaving an authorized boundary, and to enforce policy (allow, block, quarantine, encrypt, alert) before or immediately after exfiltration.

### What the interviewer is really testing

- Can you design a **multi-channel interception architecture** without assuming a single choke point?
- Can you combine **rule-based, fingerprint-based, and ML-based classification** into one coherent pipeline with an aggregated verdict?
- Do you understand the **inline-latency vs detection-completeness tradeoff** as the central anchor decision (this system's equivalent of the crawler's seed-bias problem)?
- Can you reason about **false-positive/false-negative tradeoffs** as an operational, tunable concern rather than a fixed accuracy number?
- Do you address **data residency, tenant isolation, and evidence minimization** as first-class constraints, not afterthoughts?

---

## 2. Requirements

### Functional Requirements

| Requirement | Notes |
|---|---|
| Intercept content across channels | Email, network/web egress, cloud app APIs (CASB), endpoint (clipboard/USB/print), data-at-rest |
| Extract and normalize content | Unpack archives/containers, OCR images, parse office formats, canonicalize text |
| Classify sensitivity | Regex/dictionary, Exact Data Match (EDM), Indexed Document Match (IDM/fingerprinting), ML/NLP classifiers |
| Evaluate contextual policy | User role, destination, sensitivity label, time/geo risk signals |
| Enforce a decision | Allow, warn-and-override, quarantine, block, auto-encrypt/RMS, redact, revoke share |
| Manage incidents | Correlate related events into cases; analyst triage workflow; legal-hold export |
| Provide auditability | Immutable record of every decision for SOC2 / GDPR / HIPAA / PCI-DSS |
| Continuously tune accuracy | Analyst feedback retrains classifiers and adjusts thresholds |

### Non-Functional Requirements

| Requirement | Target |
|---|---|
| Inline latency budget | P99 < 150–300ms added latency for email/proxy path; < 1–2s for interactive upload blocking |
| Scale | ~10M scan events/day for a large enterprise tenant; multi-tenant platform overall |
| Availability | Configurable fail-open vs fail-closed per channel and severity — a security decision, not just an ops one |
| Accuracy | Bound false-positive rate to avoid alert fatigue; false negatives are the harder, unbounded cost — must be reasoned about explicitly |
| Tenant isolation | One tenant's fingerprint/policy data must never be reachable by another tenant's scan path |
| Data residency | Regulated content may be legally barred from leaving its jurisdiction, even for scanning |
| Evidence minimization | Retain the least raw sensitive content necessary to support an incident, for the shortest defensible period |

---

## 3. Capacity Estimation

```
Illustrative target: single large enterprise tenant, 100,000 employees
(the platform itself is multi-tenant — see §17)

Event volume (per employee/day):
  Email (outbound + attachments)     60
  Cloud app sync/API activity        30
  Web/network uploads                10
  Endpoint (clipboard/USB/print)     10
  ────────────────────────────────────
  Total                             110  →  11,000,000 events/day

Throughput:
  11,000,000 / 86,400 ≈ 127 events/sec average
  Business-hours concentration (~8 effective hrs, 3x peak factor):
    ≈ 450–500 events/sec peak

Content volume (blended, illustrative):
  ~95% of events: median 200KB (email bodies, small attachments, form uploads)
  ~0.1% of events: "large object" class, avg 100MB (cloud file sync, datasets)
  Blended: ~11M × 200KB + 11,000 × 100MB
         ≈ 2.2 TB/day (small) + 1.1 TB/day (large) ≈ 3.3 TB/day inspected

  NOTE: unlike a crawler, most inspected content is NOT retained —
  only verdicts + evidence for flagged events are stored (see §13).

Classification worker concurrency:
  Assume ~300ms average multi-engine classification time per event
  At 500 events/sec peak: 500 × 0.3 ≈ 150 concurrent workers minimum
  →  provision ~450 workers (3× headroom, same convention as crawler fetchers)

EDM structured-data index (illustrative):
  50M sensitive records (e.g. customer SSNs/card numbers), salted-hashed
  50M × 32 bytes/hash ≈ 1.6 GB  →  fits an in-memory index (Redis/RocksDB)

IDM document fingerprint index:
  ~1M confidential documents, shingled + SimHashed (64-bit, reusing the
  Web Crawler guide's SimHash technique — see §22)
  1M × 8 bytes ≈ 8 MB core index, plus shingle postings — low 100s of MB

Flagged-event rate (illustrative, tuned target):
  ~1% of events flagged for review → 110,000/day → evidence store growth
  ~110,000 × 50KB redacted snippet ≈ 5.5 GB/day (small relative to raw traffic)
```

**Key insight:** unlike the web crawler, storage is *not* the dominant cost here — **latency budget and classification accuracy are**. The system must make a synchronous decision on the inline path within a strict SLA, using a fraction of the compute a fully offline pipeline would allow, while deliberately minimizing how much raw sensitive content it persists.

---

## 4. High-Level Architecture

```
┌───────────────┐ ┌───────────────┐ ┌───────────────┐ ┌───────────────┐
│ Email Gateway │ │ Network/Web   │ │ CASB (Cloud   │ │ Endpoint Agent│
│ (MTA / O365   │ │ Egress Proxy  │ │ App API /     │ │ (clipboard/   │
│  Graph API)   │ │ (ICAP/TLS-    │ │ reverse proxy)│ │  USB/print)   │
│               │ │  inspecting)  │ │               │ │               │
└───────┬───────┘ └───────┬───────┘ └───────┬───────┘ └───────┬───────┘
        │                 │                 │                 │
        └─────────────────┴────────┬────────┴─────────────────┘
                                    │  (also: at-rest connectors,
                                    │   batch-crawling SaaS/file shares)
                       ┌────────────▼─────────────┐
                       │  Content Extraction &     │
                       │  Normalization Pipeline   │
                       │  (unpack, OCR, canonicalize)│
                       └────────────┬─────────────┘
                                    │  (Kafka topic: extracted-content)
                       ┌────────────▼─────────────┐
                       │   Classification Engine   │
                       │  ┌──────┐┌──────┐┌──────┐ │
                       │  │Regex ││ EDM  ││ IDM/ │ │
                       │  │/Dict ││Hash  ││SimHash│ │
                       │  └──────┘└──────┘└──────┘ │
                       │  ┌──────────────────────┐ │
                       │  │ ML/NLP classifiers   │ │
                       │  │ (NER, doc-type,      │ │
                       │  │  image/perceptual)   │ │
                       │  └──────────────────────┘ │
                       └────────────┬─────────────┘
                                    │  verdicts + confidence
                       ┌────────────▼─────────────┐
                       │   Policy Engine &         │
                       │   Risk Scoring            │
                       │   (context: user, dest,   │
                       │    label, time/geo)       │
                       └────────────┬─────────────┘
                                    │
                       ┌────────────▼─────────────┐
                       │   Decision & Enforcement  │
                       │   allow / block / encrypt │
                       │   / quarantine / redact   │
                       └──────┬─────────────┬──────┘
                              │             │
                 ┌────────────▼──┐   ┌──────▼───────────┐
                 │ Incident/Case │   │  Audit Ledger     │
                 │ Mgmt + SIEM   │   │  (immutable, WORM)│
                 └───────┬───────┘   └───────────────────┘
                         │
             analyst feedback loop → retrains classifiers,
             tunes thresholds, updates allow-lists (§14)
```

**Kafka** (or an equivalent durable bus) sits between extraction, classification, and policy stages for the **out-of-band** path, giving replay and backpressure the same way it does in the crawler design. The **inline** path (email/proxy/endpoint) additionally has a synchronous call chain with a strict timeout and circuit breaker — discussed in §5.

---

## 5. Anchor Decision: Inline Enforcement vs Out-of-Band Detection

> This is the decision the rest of the design pivots on — the DLP equivalent of the crawler's seed-bias problem or the messaging guide's E2EE-first constraint. Get this wrong and every downstream component is mis-scoped.

### Inline (enforce-before-send)

The scan sits synchronously in the data path (SMTP proxy, forward/ICAP proxy, endpoint kernel hook, CASB reverse proxy) and can **block before the content leaves**.

- **Pro:** true prevention — the leak never happens.
- **Con:** every added millisecond is felt by every user, on every message; the classification pipeline becomes a hard dependency of core business workflows (sending email, saving a file). If it is down, you must choose **fail-open** (let content through — availability wins, but you're blind) or **fail-closed** (block everything — security wins, but you've taken down email for the company).

### Out-of-band (detect-and-remediate)

The scan consumes a mirrored copy (SPAN port, cloud audit-log API polling, near-real-time event stream) **after** the content has already left.

- **Pro:** zero latency impact, zero availability coupling to business-critical paths.
- **Con:** the leak has already occurred by the time you detect it. Remediation is limited to after-the-fact actions: revoke a share link, force-rotate credentials, notify the user's manager, trigger legal hold.

### The Staff-level answer: severity-tiered hybrid

```
High-confidence, high-severity signals (e.g. exact EDM match on a
customer SSN table going to a personal email domain)
  → run INLINE, fail-CLOSED, tight latency budget (simple engines only:
    regex + EDM hash lookup — no heavyweight ML inference on this path)

Medium-confidence / exploratory signals (ML classifiers, IDM fuzzy
match, contextual risk scoring)
  → run OUT-OF-BAND, feed into case management, remediate asynchronously

Low-risk channels or degraded-pipeline conditions
  → fail-OPEN with fast async catch-up scan, alert on-call
```

This mirrors the same principle as the crawler's JS-rendering split: **don't force every request through the most expensive path** — route by risk/cost, and make the routing decision explicit and auditable.

---

## 6. Component 1: Interception Points

| Channel | Mechanism | Latency profile | Notes |
|---|---|---|---|
| Email | MTA transport hook or cloud API (Graph/Gmail API) | Inline, tight budget | Attachments dominate payload size; must handle nested `.eml`/`.msg` |
| Network/web egress | Forward proxy with TLS interception, or ICAP | Inline, tight budget | Requires enterprise root CA deployed to endpoints for TLS MITM |
| Cloud apps (CASB) | Reverse proxy (SSO-integrated) or API-mode polling audit logs | Reverse proxy = inline; API-mode = out-of-band | API-mode covers unmanaged/BYOD access proxy can't intercept |
| Endpoint | Kernel driver / OS hooks (clipboard, USB mount, print spooler, local save) | Inline, sub-100ms budget | Highest-fidelity signal (pre-encryption, pre-compression) but requires agent deployment and OS-version support matrix |
| Data at rest | Scheduled connectors crawling SaaS repos, file shares, object storage | Always out-of-band (batch) | Finds *already-exposed* sensitive data (mis-shared docs) — a distinct use case from in-motion prevention |

**Why not a single choke point?** Because sensitive data leaves through many independent surfaces simultaneously, and each has a different latency contract and a different blind spot (e.g., CASB reverse-proxy mode can't see unmanaged/personal-device access; endpoint agents can't see anything once data is already synced to a cloud app the org sanctioned). A Staff-level answer treats channel coverage as a **portfolio problem**, not a single-pipe problem.

---

## 7. Component 2: Content Extraction & Normalization Pipeline

### Responsibilities

```
1. File-type sniffing by magic bytes, not extension (evasion resistance)
2. Recursive container unpacking: zip, 7z, tar, docx/xlsx/pptx (= zip),
   .eml/.msg with nested attachments, embedded OLE objects
3. OCR for images and scanned/rasterized PDFs
4. Encoding + language detection; strip formatting to canonical text
5. Metadata extraction: author, sensitivity label (e.g. MS Purview /
   Google Vault classification), creation/last-modified timestamps
```

### Decompression-Bomb Defense (the spider-trap analog)

Recursive unpacking is an attacker's easiest evasion and DoS vector — a 10KB zip that expands to 10GB nested nine layers deep can stall or crash extraction workers.

**Mitigations (directly analogous to the crawler's spider-trap defenses):**
- **Max nesting depth** (e.g., 10 layers) — abort and flag as suspicious beyond that
- **Max expansion ratio** (e.g., 100:1) — compare compressed vs decompressed size incrementally, not after full expansion
- **Max total expanded size** and **per-file time budget** — kill the extraction job, don't let it consume the worker indefinitely
- Anything that hits these limits is **treated as a policy violation in itself** (quarantine, don't silently drop) — a bomb is itself a signal of hostile intent

### Separate Fleets by Cost

OCR and archive extraction are CPU/time-expensive relative to plain-text parsing. As with the crawler's JS-rendering split:

```
Plain text / structured formats  → cheap extraction fleet
Images / scanned PDFs (OCR)      → separate fleet, own queue, own budget
Deeply nested archives           → separate fleet with strict bomb limits
```

**Senior-level answer:** mention OCR and archive handling exist.
**Staff-level answer:** describe them as separately capacity-planned fleets with independent SLAs, so a burst of large archive uploads never starves the low-latency plain-text path that most inline traffic depends on.

---

## 8. Component 3: Classification Engine

Multiple independent engines run **in parallel** on the normalized content; each emits `{verdict, confidence, matched_signal}`. Missing any one layer is a Senior-level gap; combining all of them correctly is the Staff-level signal — directly mirroring the crawler guide's two-layer (URL + content) dedup framing.

### Engine 1: Rule/Regex/Dictionary Matching

Fast, cheap, deterministic. Catches structurally regular data (SSN/credit-card-shaped numbers, keyword lists like "confidential", "NDA"). **High false-positive rate** on its own — a phone number can look like a SSN fragment — so it's rarely used as a sole blocking signal for anything but the most rigid formats (validated by checksum, e.g. Luhn check for card numbers).

### Engine 2: Exact Data Match (EDM)

Detects **known, structured** sensitive records (an actual customer table export) without shipping the raw sensitive database to every scanning node.

```
Setup: hash each sensitive record (e.g. normalized name+SSN combo)
       with a per-tenant salt → store only salted hashes in the index

Scan-time: extract candidate fields from content → apply same
           normalization + salt → hash → look up in index

Properties:
  • False negatives are the costly failure (missed real customer data)
  • False positives are cheap but still bounded via secondary field
    corroboration (e.g. require name+SSN match, not SSN alone)
  • The index never contains reversible sensitive data — a compromise
    of the scanning node does not leak the source records
```

### Engine 3: Indexed Document Matching (IDM) / Fingerprinting

Detects **unstructured** confidential documents (contracts, source code, design docs) even after partial copying, reformatting, or minor edits.

> This reuses the same **SimHash near-duplicate technique** from the Web Crawler guide's content-level dedup layer (§10 of that guide) almost unchanged: shingle the text into overlapping n-grams, compute a 64-bit fingerprint, and flag a match at Hamming distance ≤ a tuned threshold. See §22 for the cross-reference rather than re-deriving the algorithm here.

The DLP-specific twist: IDM also needs **partial-match scoring** — a document that contains a 2-paragraph verbatim excerpt of a confidential spec, embedded in an otherwise-original email, should still trigger. This requires shingling and matching at sub-document granularity, not just whole-document fingerprints.

### Engine 4: ML/NLP Classifiers

- **Contextual PII/NER models** — distinguish "here's an example SSN: 123-45-6789" (documentation) from an actual leaked value (harder than regex; reduces the false-positive load Engine 1 would otherwise generate).
- **Document-type classifiers** — source code, financial statement, medical record, resume.
- **Image classifiers / perceptual hashing** — detect photographed ID documents, screenshots of confidential dashboards (perceptual hashing survives cropping/resizing/recompression the way SimHash survives text edits).

### Aggregation

```
verdict = f(engine_1_signal, engine_2_signal, engine_3_signal, engine_4_signal)

Typical aggregation: weighted score, with EDM/IDM exact-ish matches
weighted far higher than heuristic ML confidence, and a hard override:
any single high-confidence EDM hit forces escalation regardless of
what other engines say.
```

---

## 9. Component 4: Policy Engine & Risk Scoring

Classification tells you *what* the content is; the policy engine decides *what to do about it*, using **context**:

| Signal | Examples |
|---|---|
| Sensitivity | Classification verdict + any pre-existing label (e.g. MS Purview "Highly Confidential") |
| Identity/role | Is this user authorized to handle this data class? |
| Destination | Personal webmail domain vs corporate domain; sanctioned vs unsanctioned cloud app; foreign IP/geo |
| Behavioral risk (UEBA) | Off-hours activity, impossible-travel login, recent resignation flag, unusual volume vs baseline |
| Channel | Email vs USB vs sanctioned cloud sync — same content, different risk by egress path |

```
risk_score = weighted_combination(classification_confidence,
                                   destination_risk,
                                   user_behavioral_risk,
                                   channel_risk)

policy_rules: ordered, most-specific-wins, e.g.
  IF sensitivity=PCI AND destination=personal_email → BLOCK
  IF sensitivity=Confidential AND destination=sanctioned_app → ALLOW + LOG
  IF sensitivity=Confidential AND destination=unsanctioned_app → WARN+JUSTIFY
  DEFAULT → ALLOW
```

**Staff-level nuance:** policy changes must roll out with **blast-radius control** — a canary/percentage rollout and a dry-run ("audit-only") mode before a new rule is allowed to actually block traffic, because a misconfigured rule can silently halt a business-critical workflow (e.g., blocking all outbound invoices).

---

## 10. Component 5: Malware/AV Scanning Integration

DLP (sensitive-*content*-out) and malware scanning (malicious-*content*-in) are complementary concerns that are easy to conflate. This guide deliberately does **not** re-derive multi-engine AV architecture — see the **File Upload & Malware Scanning** and **VirusTotal-style Multi-Engine Scanning Platform** guides for that (§22).

The one integration point worth calling out explicitly: **both pipelines need the same content-extraction stage** (archive unpacking, format parsing) from §7. A Staff-level design shares that extraction layer between AV and DLP scanning rather than unpacking the same nested zip twice in two independent pipelines — a straightforward but easy-to-miss efficiency and consistency win (one decompression-bomb defense policy, not two divergent ones).

---

## 11. Component 6: Decision & Enforcement Actions

| Action | When used | Notes |
|---|---|---|
| Allow | No policy match / low risk | Logged regardless, for audit completeness |
| Allow + log | Sanctioned but sensitive | Visibility without friction |
| Warn + justify (override) | Medium risk, legitimate business need plausible | User must supply a reason; captured for audit |
| Quarantine | Ambiguous / needs analyst review | Content held, sender notified of delay |
| Block + notify | High-confidence policy violation, inline path | Sender and often manager notified |
| Auto-encrypt / apply rights management | Sensitive but business-justified transfer | Integrates with e.g. Azure RMS/Purview — allows the transfer but constrains what the recipient can do with it |
| Redact + forward | Content mostly fine but contains one flagged field | Requires format-aware redaction (not just deleting text — must preserve document structure) |
| Revoke share link / rotate credentials | Out-of-band finding, already exfiltrated | Async remediation, not prevention |
| Disable account | Severe or repeated violations | Highest blast radius — reserved for confirmed high-severity cases |

**Fail-open vs fail-closed** is configured **per channel and per severity tier**, not globally — tied directly to the anchor decision in §5.

---

## 12. Component 7: Incident & Case Management / SIEM Integration

Raw flagged events are noisy; a Staff-level design **correlates events into cases** rather than surfacing each one independently to an analyst:

```
Correlation signals:
  • Same user, same destination, repeated attempts across channels
    within a time window → likely deliberate exfiltration attempt,
    not independent one-off flags
  • Same document fingerprint appearing across multiple users/channels
    → possible insider-sharing ring or compromised account fan-out
  • Volume spike vs the user's behavioral baseline
```

High-severity cases are pushed to a **SIEM/SOAR** integration for automated response playbooks (e.g., auto-disable account pending review). See the **Notification System** guide (§22) for the alert-delivery/fan-out mechanics rather than re-deriving them here.

---

## 13. Component 8: Data Stores

| Store | Contents | System | Rationale |
|---|---|---|---|
| Policy store | Versioned rules, thresholds | Low-latency KV (Redis) + durable source of truth (Postgres) | Inline path needs sub-ms reads; canary rollout needs versioning |
| EDM/IDM fingerprint index | Salted hashes, SimHash/perceptual-hash index | In-memory (Redis/RocksDB) | Same access pattern as crawler's Bloom filter — extremely high read QPS |
| Classification model registry | Versioned ML models, feature definitions | Model store (S3 + metadata DB) | Enables rollback and A/B threshold testing |
| Incident/case store | Correlated cases, analyst notes, disposition | Structured DB (Postgres/Elasticsearch for search) | Analyst workflow, reporting |
| Audit ledger | Every decision made, who/what/when, immutable | Append-only, hash-chained or object-locked (WORM) store | Compliance (SOC2/GDPR/HIPAA/PCI-DSS) — must survive even a compromised admin account |
| Evidence store | Redacted/tokenized snippets for flagged events only | Encrypted, access-controlled, retention-limited | Data-minimization: not a copy of all inspected traffic |

**Key design point, unlike the crawler:** this system should **minimize** what it retains. The crawler's storage layer is built to keep everything; a DLP system's evidence store should keep the least raw sensitive content necessary to support a case, for the shortest legally defensible period, with the audit *decision* record (which is metadata, not content) being the thing that's kept indefinitely.

---

## 14. Component 9: Feedback Loop & Model Tuning

This is the DLP-specific equivalent of the crawler's adaptive-freshness/Poisson-process strategy (§17 of that guide): a continuous drift-correction loop rather than a fixed accuracy number.

```
Analyst disposition on each case (true positive / false positive)
  feeds back into three places:

  1. Allow-list additions — recurring benign matches (e.g. a template
     contract that always trips IDM) get added to a suppression list
  2. Classifier retraining — labeled examples improve ML precision/recall
     over time; retrain on a schedule, not ad hoc
  3. Threshold auto-tuning — per-policy confidence thresholds adjust to
     hold the false-positive rate within a target band, the same way
     the crawler adjusts recrawl interval λ based on observed change rate
```

**Staff-level framing:** false-positive rate and false-negative rate trade off against each other and against **analyst throughput** (a human review queue that grows faster than analysts can clear it is a real operational failure mode, not just an accuracy metric). Treat the target FP rate as a tunable operating point chosen jointly with the SOC team's capacity, not a number to be minimized in isolation.

---

## 15. CAP / PACELC Positioning

General framework: see the **CAP/PACELC Theorem** guide (§22). Applied per component here:

| Component | Position | Reasoning |
|---|---|---|
| Inline policy decision path | **CP-leaning (PC)** | A high-severity block decision must use current policy, not stale — but must have a circuit breaker to a safe default rather than hang indefinitely under partition |
| EDM/IDM fingerprint index | **AP/EL**, tunable per index | Short staleness (seconds) after a policy/fingerprint update is acceptable for most tiers; a "kill switch" fast path bypasses this for emergency revocations |
| Policy store replication to edge nodes | **AP/EL** | Bounded staleness SLA (e.g. <60s propagation); emergency updates use a separate strongly-consistent path |
| Audit ledger (within region) | **CP** | Compliance requires no silent loss and no fork of the record — consistency wins even under partition |
| Audit ledger (cross-region read replicas) | **AP** | Read replicas for reporting/dashboards can lag; the authoritative write path stays CP |
| Incident/case store | **AP/EL** | Standard CRUD workload; eventual consistency across replicas is fine |
| Classification/ML inference | **AP/EL** | A slightly stale model version producing a marginally different confidence score is an acceptable tradeoff for availability |

---

## 16. Failure Modes & Mitigations

| Failure | Symptom | Detection | Mitigation |
|---|---|---|---|
| Classification pipeline down | Inline path has no verdict to act on | Health check / circuit breaker trips | Fail-open or fail-closed per pre-configured channel/severity policy (§5); never block indefinitely |
| Decompression bomb | Extraction worker stalls/OOMs | Expansion-ratio and time-budget monitors | Hard nesting/size/time limits; treat limit-hit as a violation itself, not a silent drop |
| Adversarial evasion (encoding change, homoglyphs, steganography) | Sensitive content passes undetected | Periodic red-team/purple-team testing; drift in FN rate on known test corpus | Normalize aggressively before classification; treat evasion techniques themselves as a detectable signal |
| EDM/IDM index staleness after a breach | New sensitive record set not yet fingerprinted | Time-to-fingerprint SLA monitoring | Fast-path re-fingerprinting pipeline with priority queue for newly-flagged sensitive datasets |
| OCR/archive fleet backlog | Large-file/image scan queue grows | Consumer lag on that fleet's queue | Scale fleet independently; degrade gracefully to out-of-band for that content class rather than blocking inline |
| Policy misconfiguration causing mass block | Sudden spike in blocked legitimate traffic | Block-rate anomaly alarm vs baseline | Canary/percentage rollout + audit-only dry-run mode before any rule can enforce (§9) |
| Alert storm overwhelms incident store/analysts | Case queue depth spikes, SLA breach | Queue depth monitoring | Correlation/deduplication into cases (§12); auto-triage low-confidence cases; back-pressure into async-only mode |
| Cross-tenant data leakage in shared index | One tenant's fingerprint/policy visible to another | Access-pattern auditing, tenant-scoped query tracing | Hard tenant partitioning of indices and policy stores (§17), never a shared global fingerprint namespace |
| Clock skew across scanning nodes | Correlation window logic misfires | Case-correlation false merges/splits | Logical clocks / bounded NTP skew tolerance in correlation windows |

---

## 17. Scalability & Sharding

### Multi-Tenant Sharding

```
Shard by tenant_id for:
  • Fingerprint indices (EDM/IDM) — hard isolation, never shared
  • Policy stores — per-tenant rule sets
  • Incident/case stores — per-tenant analyst workspaces

Rationale: a fingerprint index is itself sensitive (it encodes what a
company considers confidential, and salted hashes of real customer
data). Cross-tenant exposure of the index is a breach in its own right,
not just a bug — this is the DLP-specific analog of the crawler's
"one coordinator becomes a bottleneck" problem, except the failure
mode here is a security incident, not a performance one.
```

### Geo-Sharding

```
shard_id = hash(tenant_region)

Each region owns its own extraction, classification, and index
infrastructure for tenants domiciled there — driven by data residency
requirements (§18), not just load distribution.
```

### Channel/Cost-Based Fleet Scaling

Same principle as the crawler's fetcher-vs-parser split: scale each fleet independently by its own bottleneck resource.

| Fleet | Bottleneck | Scaling lever |
|---|---|---|
| Plain-text extraction | I/O | Horizontal, stateless |
| OCR | CPU/GPU, expensive | Separate pool, own budget/queue |
| ML inference | GPU | Batch-friendly, autoscale on queue depth |
| Rule/regex engine | CPU, cheap | Runs inline, minimal footprint |
| EDM/IDM index lookups | Memory-bound | Scale index shard count with dataset size |

---

## 18. Data Residency & Jurisdictional Constraints

> A Staff-level differentiator, structurally similar to the crawler guide's seed-bias callout: a constraint that must be designed in from the start, not patched on later.

Some tenants' content is legally barred from leaving its jurisdiction — even for scanning by a shared multi-region service. Implications:

- **Region-local classification**: EU tenant content is extracted and classified entirely within EU infrastructure; no raw content crosses the border, not even transiently in a queue.
- **Metadata-only cross-region aggregation**: global reporting, threat intelligence, and platform-wide model improvement work from **verdicts and anonymized metadata**, never raw content, when aggregating across regions.
- **Per-tenant residency policy as a first-class routing key**, not an exception path — the ingestion layer (§6) must route content to the correct regional pipeline before any extraction happens.

---

## 19. Senior vs Staff Answer Differentiators

### Senior-level expectations

- Correctly identify the major components (interception, extraction, classification, policy, enforcement)
- Mention regex/keyword matching and basic ML classification
- Mention encryption/blocking as enforcement actions
- Handle the obvious failure mode (scanning service down → fail-open or closed)
- Provide basic capacity math

### Staff-level expectations (additional)

| Topic | What Staff-level adds |
|---|---|
| Anchor decision | Proactively frame inline-vs-out-of-band as a severity-tiered hybrid, not a binary choice |
| Classification | All three layers (rule, EDM, IDM/fingerprint) plus ML, with an explicit aggregation strategy and FP/FN reasoning |
| Extraction pipeline | Decompression-bomb defense as its own hardened concern; separate fleets by cost (OCR, archive, ML) |
| Policy engine | Context beyond content sensitivity: destination, behavioral risk, channel; canary rollout for blast-radius control |
| Incident management | Event correlation into cases, not raw alert firehose |
| Storage | Explicit data-minimization stance — retain the least raw content, longest only what compliance requires |
| Tenant isolation | Fingerprint/policy cross-tenant leakage framed as a security incident, not a performance concern |
| Data residency | Region-local pipelines as a routing-time decision, not a post-hoc filter |
| Feedback loop | Continuous threshold/model tuning tied to analyst capacity, not a static accuracy target |
| CAP/PACELC | Explicit per-component positioning, including the split between the strongly-consistent "kill switch" path and the eventually-consistent default policy propagation |

### The single most important Staff differentiator

**Naming the false-negative cost explicitly, and refusing to present accuracy as a single number.** A Senior answer says "we use ML to detect PII with high accuracy." A Staff answer says: false positives cost analyst time and user trust; false negatives cost an actual data breach; the two are asymmetric, the target operating point is a business decision tied to analyst capacity and the tenant's risk tolerance, and the system must expose that tradeoff as a tunable, not bury it inside a model.

---

## 20. Interview Time Allocation

For a 45-minute system design session:

| Phase | Time | Focus |
|---|---|---|
| Requirements & anchor decision | 7 min | Inline vs out-of-band; fail-open/closed; confirm scale target |
| Capacity estimation | 5 min | Events/sec, latency budget, index sizing |
| High-level architecture | 5 min | Draw all interception points converging into one pipeline |
| Extraction & classification (deep dive) | 12 min | Decompression-bomb defense, EDM/IDM/ML aggregation |
| Policy engine & enforcement | 7 min | Context signals, canary rollout, action matrix |
| Storage & incident management | 3 min | Data-minimization stance, correlation into cases |
| Failure modes | 5 min | Fail-open/closed, tenant isolation, alert storms |
| Data residency / feedback loop (if time) | 3–5 min | Region-local routing, threshold tuning |

**What to cut if short on time:** deep ML model architecture and full storage schema detail. **Never cut:** the anchor inline/out-of-band decision, the classification aggregation strategy, or tenant isolation.

---

## 21. Quick-Reference Cheatsheet

```
KEY ALGORITHMS
──────────────
Structured match:   Exact Data Match (EDM)  |  salted hash lookup  |  no reversible data at the edge
Unstructured match: Indexed Document Match  |  SimHash (reused from Web Crawler dedup)  |  Hamming ≤ threshold
Contextual PII:     ML/NER classifiers      |  distinguishes real leaks from documentation examples
Image match:        Perceptual hashing      |  survives crop/resize/recompression
Bomb defense:       Nesting depth + expansion ratio + total size + time-budget caps

KEY NUMBERS (illustrative, 100K-employee tenant)
─────────────────────────────────────────────────
11M events/day  →  ~127/sec avg, ~450–500/sec peak
~300ms multi-engine classify time → ~450 workers w/ 3× headroom
EDM index: 50M records × 32B hash ≈ 1.6 GB
IDM fingerprint index: 1M docs × 8B ≈ 8 MB core index
~1% flagged rate → ~110K cases/day → ~5.5 GB/day evidence (not raw traffic)

KEY SYSTEMS
───────────
Interception:    Email gateway, ICAP/forward proxy, CASB, endpoint agent, at-rest connectors
Extraction bus:  Kafka (out-of-band path); synchronous call chain (inline path)
Fingerprint idx: Redis/RocksDB (EDM hashes, SimHash/perceptual index)
Policy store:    Redis (hot, edge) + Postgres (source of truth, versioned)
Audit ledger:    WORM / hash-chained append-only store
Incident store:  Postgres/Elasticsearch (case search + analyst workflow)

CAP/PACELC DECISIONS
─────────────────────
CP:  Inline block decision path, audit ledger write path (in-region)
AP:  Fingerprint index reads, policy propagation to edge, incident store, cross-region audit reads
Separate strongly-consistent "kill switch" path bypasses normal AP policy propagation

ANCHOR DECISION
───────────────
High-confidence/high-severity  → INLINE, fail-CLOSED, cheap engines only (regex+EDM)
Medium-confidence/exploratory  → OUT-OF-BAND, async remediation
Degraded pipeline              → fail-OPEN + fast async catch-up scan

STAFF-LEVEL SIGNALS TO HIT
───────────────────────────
✓ Severity-tiered inline/out-of-band hybrid, not a binary choice
✓ Three-plus-layer classification (rule, EDM, IDM, ML) with explicit aggregation
✓ Decompression-bomb defense as a hardened, first-class concern
✓ Tenant isolation of fingerprint/policy data framed as a security boundary
✓ Data-minimization storage stance (don't keep what you don't need)
✓ Data residency as a routing-time decision, not a post-hoc filter
✓ FP/FN tradeoff named explicitly, tied to analyst capacity
✓ Canary/audit-only rollout for policy changes (blast-radius control)
✓ Event correlation into cases, not a raw alert firehose
✓ Explicit CAP/PACELC per component, including a separate strongly-consistent kill-switch path
```

---

## 22. Related Guides

- **File Upload & Malware Scanning** — the file-upload-specific scanning path and AV integration this guide's §10 deliberately points to rather than re-deriving.
- **VirusTotal-style Multi-Engine Scanning Platform** — multi-engine aggregation architecture for malware verdicts; the same aggregation *pattern* used here for classification engines in §8.
- **Web Crawler** — source of the SimHash near-duplicate technique reused for IDM fingerprinting (§8), and the spider-trap pattern reused for decompression-bomb defense (§7).
- **CAP/PACELC Theorem** — general framework referenced in §15 rather than re-derived.
- **Circuit Breaker Pattern** — the mechanism underlying the fail-open/fail-closed behavior on the inline path (§5, §16).
- **Apache Kafka** — the durable bus used for the out-of-band extraction/classification pipeline (§4).
- **Notification System** — alert-delivery and fan-out mechanics for incident notifications (§12).

---

*Guide compiled for Senior & Staff-level system design interview preparation.*
*Topics: Data Loss Prevention, File Scanning, Content Classification, Fingerprinting, Policy Engines, Multi-Tenant Security Systems.*
