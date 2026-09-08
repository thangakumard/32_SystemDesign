# Secure Key Vault Store — System Design Interview Guide
### Senior & Staff Engineer Level  (Azure Key Vault / HashiCorp Vault / AWS KMS-style)

---

## Table of Contents

1. [Problem Statement & Scope](#1-problem-statement--scope)
2. [Requirements](#2-requirements)
3. [Capacity Estimation](#3-capacity-estimation)
4. [High-Level Architecture](#4-high-level-architecture)
5. [Component 1: Client SDK & API Gateway](#5-component-1-client-sdk--api-gateway)
6. [Component 2: AuthN — Identity Verification](#6-component-2-authn--identity-verification)
7. [Component 3: Access Policy Engine (AuthZ)](#7-component-3-access-policy-engine-authz)
8. [Component 4: Cryptographic Core — Key Hierarchy & HSM Cluster](#8-component-4-cryptographic-core--key-hierarchy--hsm-cluster)
9. [Component 5: Secrets / Keys / Certificates Service](#9-component-5-secrets--keys--certificates-service)
10. [Component 6: Key Rotation & Certificate Lifecycle](#10-component-6-key-rotation--certificate-lifecycle)
11. [Component 7: Metadata & Storage Layer](#11-component-7-metadata--storage-layer)
12. [Component 8: Audit Logging Service](#12-component-8-audit-logging-service)
13. [Component 9: Soft-Delete, Purge Protection & Backup](#13-component-9-soft-delete-purge-protection--backup)
14. [CAP Theorem Positioning](#14-cap-theorem-positioning)
15. [Failure Modes & Mitigations](#15-failure-modes--mitigations)
16. [Scalability & Sharding](#16-scalability--sharding)
17. [Multi-Region Replication & Key Ceremony](#17-multi-region-replication--key-ceremony)
18. [Senior vs Staff Answer Differentiators](#18-senior-vs-staff-answer-differentiators)
19. [Interview Time Allocation](#19-interview-time-allocation)
20. [Quick-Reference Cheatsheet](#20-quick-reference-cheatsheet)
21. [Related Guides](#21-related-guides)

---

## 1. Problem Statement & Scope

A **key vault** is a centralized, hardware-backed service for storing and managing **secrets** (passwords, connection strings, tokens), **keys** (symmetric/asymmetric material used for encrypt/decrypt/sign/verify), and **certificates** (X.509 lifecycle). It is not just an encrypted database — its defining property is that it lets applications *use* cryptographic material without ever *possessing* it, and it gives security teams a single, auditable choke point for every access to sensitive material in an organization.

### What the interviewer is really testing

- Do you understand **envelope encryption** and the **key hierarchy** well enough to explain why a compromised application server still can't recover the root key?
- Can you reason about **fail-closed vs fail-open** as a deliberate security decision, not an oversight?
- Do you treat **AuthN, AuthZ, and audit** as first-class distributed-systems components with their own consistency and availability tradeoffs — not an afterthought bolted onto a CRUD API?
- Can you make **explicit CAP decisions**, and justify why this system's default leans CP where a typical service would lean AP?

---

## 2. Requirements

### Functional Requirements

| Requirement | Notes |
|---|---|
| Store and retrieve secrets (opaque key-value blobs) | Passwords, connection strings, API tokens |
| Create, use, and rotate cryptographic keys | Private key material never leaves the vault |
| Manage X.509 certificate lifecycle | Issuance, renewal, binding to consumers |
| Fine-grained access control per object/vault | RBAC + attribute-based conditions |
| Full audit trail of every access | Who, what, when, from where, allow/deny |
| Versioning with rollback | Every write is immutable; "current" is a pointer |
| Soft-delete with recoverable window | Protect against accidental/malicious deletion |

### Non-Functional Requirements

| Requirement | Target |
|---|---|
| Confidentiality | Root/master key material never exists in plaintext outside an HSM boundary |
| Availability (data plane read) | 99.9%+, regional failover within minutes |
| Latency | p99 < 50ms for cached/metadata reads; p99 < 200ms for HSM-bound crypto ops |
| Auditability | Tamper-evident, immutable, exported to SIEM |
| Compliance | FIPS 140-2/3, SOC 2, support for BYOK/HYOK |
| Multi-tenancy isolation | One tenant's compromise must not expose another's key material |

---

## 3. Capacity Estimation

```
Target: 100,000 active vaults (tenants)
Platform peak: 500,000 requests/sec (data plane: get/set/encrypt/decrypt/sign)

Traffic mix:
  ~90% metadata/policy-cacheable reads (GET secret, list, check-exists)
  ~10% writes + true HSM-bound crypto ops (sign, encrypt, decrypt, wrap/unwrap)

True HSM-bound ops (cannot be cached — must touch the crypto core):
  ~15% of total traffic ≈ 75,000 ops/sec

HSM partition throughput (RSA-2048 sign ≈ 1,000–2,000 ops/sec per HSM card,
AES symmetric ops much higher):
  75,000 / 1,500 ≈ 50 HSM partitions minimum
  → provision ~150 (3× headroom + N+2 redundancy per cluster)

Audit log throughput (every request emits a durable event):
  500,000 events/sec × ~1KB/event ≈ 500 MB/s
  → Kafka, ~50 partitions at 10MB/s each

Metadata store (vault, object, version, ACL rows):
  100,000 vaults × ~500 objects/vault × ~5 versions avg ≈ 250M rows
  → trivial row count for a wide-row store; sized by throughput, not volume

Encrypted payload storage (secrets/keys/certs, typically <25KB each):
  250M objects × 25KB ≈ 6.25 TB
```

**Key insight — the inverse of a typical high-throughput system:** unlike a crawler or feed system, **storage volume is not the bottleneck here.** The two dominant constraints are (1) **HSM crypto throughput**, which is orders of magnitude more expensive per operation than a disk write, and (2) **durable audit-write throughput**, because a security-critical mutation that isn't safely logged is treated as a failed operation, not a fire-and-forget side effect.

---

## 4. High-Level Architecture

```
                              ┌────────────────────┐
                              │   Client / App       │
                              │  SDK + mTLS +         │
                              │  Managed Identity     │
                              └──────────┬────────────┘
                                         │ HTTPS + OAuth2/OIDC token
                              ┌──────────▼────────────┐
                              │   API Gateway /         │
                              │   Data-Plane Front Door │
                              │   (per region)          │
                              └──────────┬────────────┘
                     ┌────────────────────┼────────────────────┐
                     │                    │                    │
           ┌─────────▼──────┐   ┌─────────▼─────────┐  ┌───────▼────────┐
           │  AuthN Service   │   │  Access Policy      │  │  Rate Limiter /│
           │  validate token  │   │  Engine (AuthZ)      │  │  Circuit       │
           │  + mTLS chain    │   │  RBAC + ABAC         │  │  Breaker       │
           └─────────┬──────┘   └─────────┬─────────┘  └───────┬────────┘
                     │                    │                    │
                     └────────────────────┼────────────────────┘
                                          │  (all three must pass — fail closed)
                              ┌──────────▼────────────┐
                              │   Vault Object Router    │
                              │   shard by vault_id       │
                              └──────────┬────────────┘
                    ┌─────────────────────┼─────────────────────┐
                    │                     │                     │
          ┌─────────▼──────┐   ┌──────────▼─────────┐  ┌────────▼───────┐
          │  Secrets/Keys/   │   │  Cryptographic       │  │  Metadata &     │
          │  Certs Service   │◄──┤  Core (HSM Cluster)   │  │  Version Store  │
          │  lifecycle,      │   │  Root → KEK → DEK      │  │  (wide-row,     │
          │  versioning      │   │  hierarchy              │  │  partition by   │
          └─────────┬──────┘   └──────────┬─────────┘  │  vault_id)      │
                    │                     │             └────────┬───────┘
                    └─────────────────────┼─────────────────────┘
                                          │
                              ┌──────────▼────────────┐
                              │   Audit Log Service      │
                              │   fail-closed on mutate,  │
                              │   WORM store, hash-chained│
                              └───────────────────────────┘
```

Unlike a typical CRUD service, **every request fans out through three independent gates — AuthN, AuthZ, and rate-limiting — before it ever reaches the object it's asking about.** None of these gates are optional or best-effort; a failure in any one of them must resolve to a denial, not a pass-through.

---

## 5. Component 1: Client SDK & API Gateway

### What it is

A per-region, stateless front door that terminates mTLS, performs coarse request validation, and forwards to backend services. The SDK embedded in client applications implements retry with exponential backoff and jitter, and a **circuit breaker** so a struggling vault doesn't get hammered by thousands of retrying callers during a partial outage.

### Why per-region

Data residency requirements (many customers legally require secrets to never leave a jurisdiction), latency, and **blast-radius containment** — a regional front-door failure should not cascade into a global outage.

### Identity flow, not static credentials

Clients authenticate with a short-lived **OAuth2/OIDC token from a platform identity provider** (Azure Managed Identity, AWS IAM role, or SPIFFE/SPIRE for on-prem/hybrid) plus mTLS, rather than a long-lived static API key. This solves the classic "chicken-and-egg" problem — you need a credential to fetch a credential — by letting the compute platform itself attest to workload identity.

### SDK-Side Caching Rule

The SDK may cache **non-sensitive metadata** (object existence, version numbers, tags) with a short TTL. It must **never cache raw secret plaintext beyond the scope of the calling request** unless the caller explicitly opts into a bounded, short TTL cache — because every cached secret is a second place an attacker can steal it from. This constraint reappears in the CAP discussion below.

---

## 6. Component 2: AuthN — Identity Verification

| Mechanism | Pros | Cons |
|---|---|---|
| Managed/Workload Identity (OAuth2/OIDC token) | No credential to leak; short-lived; platform-attested | Requires the platform's identity infrastructure to be up |
| mTLS client certificate | Strong, works off-platform, mutual verification | Certificate distribution/rotation overhead |
| Static API key | Simple, works anywhere | Long-lived; a leaked key has no automatic expiry; hardest to audit |
| SAS-style scoped token | Fine-grained, time-boxed, delegable | Still a bearer credential; must be transmitted and stored carefully |

**Default posture:** require Managed Identity or mTLS for production traffic; support static keys only for legacy/break-glass paths, with alerting on their use. AuthN tokens are deliberately short-lived (5–15 minutes) so a compromised token has a small, bounded window of usefulness — this forces continuous liveness with the identity provider rather than a one-time check.

---

## 7. Component 3: Access Policy Engine (AuthZ)

Two distinct layers, often confused by Senior-level candidates:

| Layer | Governs | Example |
|---|---|---|
| **Management plane RBAC** | Who can create, delete, or configure a *vault itself* | "Can this identity create a new vault in this subscription?" |
| **Data plane access policy** | Who can GET/SET/DELETE a *specific object* inside a vault | "Can this identity read the `db-connection-string` secret?" |

### Policy Design Principles

- **Deny-by-default, explicit allow** — an object with no policy is inaccessible, not open.
- **Policies are themselves versioned, auditable objects**, stored in the metadata store — never inside the object they protect, to avoid a circular dependency (you shouldn't need to decrypt a secret to find out whether you're allowed to read it).
- **ABAC layered on RBAC** — conditions on network origin (private endpoint only), time-of-day, or purpose tags, for organizations that need more than role membership.
- **Just-in-time (JIT) access** — temporary elevated permissions with automatic expiry and mandatory approval, rather than standing privilege that sits unused (and unnoticed) for months.
- **Break-glass procedure** — emergency access requires dual control (two distinct approvers) and generates a maximum-severity audit flag, so it's usable in a real emergency but never quietly abusable.

### Staff-Level Gotcha: The Read-After-Revoke Problem

If a policy change (a revoked grant) hasn't propagated to every regional replica, a request routed to a stale replica can return "allow" for an identity that was just revoked. This is a security incident, not a UX inconvenience — it's the AuthZ analog of stale DNS or a stale robots.txt cache, except the cost of staleness is categorically higher.

**Fix:** access-policy reads for privileged operations must go through a **strongly consistent, quorum-backed path** (not an arbitrary nearest replica), and the SDK/gateway-side policy cache TTL is measured in single-digit seconds, not minutes. This is the single clearest place where this system's CAP posture diverges from a typical read-heavy service.

---

## 8. Component 4: Cryptographic Core — Key Hierarchy & HSM Cluster

This is the heart of the vault — the component every other piece of the system exists to protect.

### The Key Hierarchy

```
Hardware Root Key  (generated + held inside a FIPS 140-2/3 HSM, non-exportable)
        │  wraps
        ▼
Vault Master Key (KEK)  — one per tenant/vault, itself wrapped by the root
        │  wraps
        ▼
Data Encryption Key (DEK)  — generated fresh per object write
        │  encrypts
        ▼
Actual secret / key / certificate payload  (ciphertext at rest)
```

### Envelope Encryption — Write Path

```
1. Client sends plaintext secret over mTLS
2. Service generates a fresh, random DEK (AES-256)
3. Payload encrypted locally with the DEK (AES-256-GCM)
4. DEK sent to the HSM to be "wrapped" (encrypted) by the vault's KEK
5. Raw DEK zeroed from process memory immediately after wrapping
6. {ciphertext, wrapped_DEK} persisted to storage — the raw DEK never touches disk
```

### Envelope Encryption — Read Path

```
1. Fetch {ciphertext, wrapped_DEK} from storage
2. Send wrapped_DEK into the HSM boundary to be unwrapped by the KEK
   (the KEK itself never leaves the HSM — only the resulting raw DEK does,
   over an authenticated in-process/in-memory channel)
3. Decrypt payload locally with the raw DEK
4. Return plaintext to the caller over mTLS
5. Zero the raw DEK from memory immediately
```

**The core invariant:** any operation that touches the KEK happens *inside* the HSM boundary. Only wrapped (already-encrypted) key material and short-lived, per-operation DEKs ever cross that boundary — the KEK and root key never do, under any code path, including error/debug paths.

### Keys vs. Secrets — A Fundamental Distinction

For a **Secret**, the plaintext is ultimately returned to the caller (that's the point — it's a password or connection string an application needs to use directly).

For a **Key** object, the private key material **never leaves the vault, even to its legitimate owner.** Every operation — sign, verify, encrypt, decrypt, wrapKey, unwrapKey — is a remote call *into* the vault: the caller sends data to be operated on and receives only the result. This is the feature that distinguishes a true key-management service from a glorified encrypted key-value store, and it's a detail Senior-level candidates frequently miss.

### Threshold Cryptography — Unsealing Without a Single Point of Compromise

On a cold start (HSM cluster reboot, disaster recovery), the root key material must be reconstructed. It is never held complete by any single machine or operator. Using **Shamir's Secret Sharing**, the root key is split into N shares (e.g., 5) distributed to independent key-holders, and reconstruction requires a **quorum** (e.g., 3-of-5) to "unseal" the cluster — the pattern used in production by HashiCorp Vault and standard HSM key-ceremony practice. No single compromised operator, and no single stolen laptop, can ever reconstruct the root key alone.

### HSM Cluster Topology

A pool of physical (or cloud-attested virtual — e.g., AWS CloudHSM, Nitro-enclave-backed) HSMs behind a partition-aware router. **Sharded by `vault_id`**, so a tenant's KEK always resolves to the same partition — this avoids cross-partition key synchronization on the hot path and bounds blast radius: compromise of one partition affects only the tenants pinned to it.

---

## 9. Component 5: Secrets / Keys / Certificates Service

| Object Type | Purpose | Material Exposure | Typical Rotation |
|---|---|---|---|
| Secret | Arbitrary opaque value (password, connection string, token) | Plaintext returned to authorized caller | Manual or scheduled, 30–90 days |
| Key | Cryptographic key used via operations, not retrieved | Private material **never** leaves the vault | Automated, 90–365 days, with grace overlap |
| Certificate | X.509 cert + associated private key | Private key never leaves; cert (public) may be exported | Auto-renewed N days before expiry |

Every write creates an **immutable new version**; the "current version" is a pointer moved atomically via compare-and-swap on the metadata row. This gives free rollback, a natural audit trail, and lets consumers pin to a specific version for reproducibility.

**Certificates carry extra lifecycle complexity:** integration with an internal CA or ACME provider, automatic re-issuance ahead of expiry, and — critically — a **notification/webhook mechanism** so dependent resources (a load balancer's TLS binding, a service mesh sidecar) can pick up the renewed certificate without manual intervention. A renewal that "succeeds" in the vault but never reaches the consumer is a common, painful real-world outage pattern.

---

## 10. Component 6: Key Rotation & Certificate Lifecycle

### Dual-Version Validity Window

Rotation is never an instant cutover. Old and new key versions are **both valid for a configured grace period**: new writes use the new key, but data already encrypted under the old key remains decryptable until every consumer has migrated. Skipping this window turns a routine rotation into a synchronized "flag day" outage across every consuming service.

### Cache Invalidation on Rotation

Automated rotation must call back into consuming services (event/webhook) so cached references get refreshed — this is fundamentally a distributed cache-invalidation problem, the same shape as staleness handling in any replicated read-heavy system.

### The Silent-Failure Trap (Staff-level gotcha)

A certificate renewal that fails silently — logged but not alerted — surfaces weeks later as an unplanned TLS outage when the old certificate expires. **Renewal failure must be treated as page-worthy at T-minus-30/14/7/1 days**, not merely logged, because "the renewal job ran" and "the renewal job succeeded" are different facts that get conflated in weaker designs.

---

## 11. Component 7: Metadata & Storage Layer

| Layer | What | System | Rationale |
|---|---|---|---|
| Metadata store | Vault, object, version, current-version pointer | Wide-row store (Cassandra-style), partitioned by `vault_id` | High write throughput; linearizable CAS on the version pointer |
| Encrypted payload store | `{ciphertext, wrapped_DEK}` blobs | Object storage or wide-row, replicated across AZs | Payloads are inert without the KEK — safe to replicate eventually-consistently |
| Access policy store | RBAC/ABAC rules, versioned | Strongly consistent store (e.g., Raft-replicated) | Staleness here is a security incident, not an inconvenience |
| Audit store | Immutable, hash-chained event log | Kafka → WORM/immutable storage | Tamper-evidence and durability outrank query flexibility |

**The core security invariant of the whole storage design:** the metadata and payload stores **never contain plaintext secret material or raw key material** — only ciphertext, wrapped DEKs, and descriptive metadata. Compromising the storage layer alone, without also compromising the HSM boundary, yields nothing decryptable. This "two independent compromises required" property is worth stating explicitly in an interview — it's the payoff of the entire key-hierarchy design.

---

## 12. Component 8: Audit Logging Service

### Fail-Closed vs. Fail-Open — A Deliberate Split

- **Mutating / control-plane events** (create, delete, rotate, policy change): if the audit write fails, **the operation itself must fail.** The audit record is part of the transaction, not a side effect — an unlogged privileged mutation is worse than a temporarily unavailable vault.
- **High-volume read events** (routine GETs): full fail-closed on every read would make audit-pipeline health a single point of failure for the entire platform's availability. These are logged **asynchronously and best-effort**, buffered through Kafka, with alerting on pipeline lag and a bounded backpressure signal to callers if the backlog grows dangerously large.

Presenting this as a deliberate, reasoned split — not a blanket "always fail closed" or "always fail open" rule — is a clear Staff-level signal.

### Tamper Evidence

Each audit record is **hash-chained** to the previous record (Merkle-chain style), so any retroactive edit breaks the chain. The chain is exported to WORM (write-once-read-many) storage, and the chain's root hash can be periodically anchored to an external timestamping service for non-repudiation.

**Rule:** an audit record captures the operation, actor, object identifier, source, and allow/deny result — **never the secret payload itself.**

---

## 13. Component 9: Soft-Delete, Purge Protection & Backup

### Soft-Delete

A DELETE marks the object (or entire vault) as deleted, but retains ciphertext and metadata for a recovery window (commonly ~90 days). The object becomes inaccessible via the normal API and is recoverable only via an explicit, separately authorized Recover call.

### Purge Protection

An optional, often mandatory-for-production flag that prevents **even an authorized principal** from immediately hard-deleting during the recovery window. This protects against both a malicious insider and a compromised-but-legitimate credential performing a fast "smash and grab" delete — the recovery window can't be bypassed just because you have delete permission.

### Backup / Restore

Exports are **encrypted bundles**, still wrapped by the vault's KEK (or re-wrapped under a customer-supplied export key) — never a plaintext export. Restoring either re-establishes the link into the original key hierarchy or requires re-wrapping the payload under the target vault's KEK.

---

## 14. CAP Theorem Positioning

Most distributed systems default to AP with a handful of named CP exceptions. **A key vault inverts that default** — most of its security-critical components deliberately choose CP, with AP reserved for the pieces where staleness is genuinely harmless. This inversion, and the ability to justify it per component, is the central CAP conversation for this design.

| Component | CAP Choice | Reasoning |
|---|---|---|
| Access Policy Engine (AuthZ decision) | **CP** | Serving a stale "allow" after revocation is a security incident. Under partition: fail closed (deny), never guess. |
| Cryptographic Core (HSM cluster) | **CP** | Key state must never diverge across replicas. Sharded to one authoritative partition per vault — no multi-master key mutation. |
| Audit log — mutating/control-plane events | **CP (fail-closed)** | The write must not be considered complete unless the audit record is durably recorded. |
| Audit log — high-volume read events | **AP / EL** | Fail-closed on every GET would make audit-pipeline health a platform-wide SPOF; buffer async, alert on lag. |
| Access-policy store (RBAC/ABAC rules) | **CP** | Same reasoning as the AuthZ decision itself — the store backing it can't be the source of the staleness. |
| Metadata store — version pointer | **CP** (linearizable CAS) | Two writers must not both believe they hold the "current" version. |
| Metadata store — descriptive fields (tags, labels) | **AP / EL** | A stale tag is a cosmetic issue, not a security one. |
| Encrypted payload/blob store | **AP** | Ciphertext is inert without the KEK; a stale-but-valid version is harmless to serve. |
| Rate limiter / circuit breaker state | **AP** | Approximate throttling under partition is a cost/availability tradeoff, not a security one. |

---

## 15. Failure Modes & Mitigations

| Failure | Symptom | Detection | Mitigation |
|---|---|---|---|
| HSM partition failure | Vault for the affected shard becomes unavailable | HSM heartbeat/health check failure | N+2 redundancy per partition; automatic intra-partition failover; the vault is intentionally unavailable rather than silently insecure |
| Stale ACL replica served | Revoked identity still receives "allow" | Access-after-revoke security alert | Policy reads via a quorum/strongly-consistent path; single-digit-second policy cache TTL |
| Audit pipeline backlog | Kafka consumer lag on the audit topic | Consumer-lag monitoring | Control-plane events: block/fail-closed; read events: async catch-up with alerting and bounded backpressure |
| Rotation without grace period | Mass decrypt failures immediately after rotation ("flag day") | Spike in decrypt-failure rate post-rotation | Dual-version validity window; staged/canary rollout |
| Certificate renewal silently fails | Certificate expires unnoticed → downstream TLS outage | Expiry-window alerting, not just renewal-attempt logging | Page at T-minus 30/14/7/1 days; treat renewal failure as an incident |
| Insider threat / rogue admin | Unauthorized bulk secret access or export | Anomaly detection on access volume, timing, object scope | No single admin holds the root key (Shamir threshold); dual control on high-risk ops; JIT access with automatic expiry |
| SDK-side cache poisoning | Client acts on stale or tampered cached metadata | Integrity check on cached responses | Cache only non-sensitive fields; sign cache-eligible responses; short TTL |
| Regional outage | Vault unreachable in the affected region | Regional health probes | Multi-region deployment with pre-established key-ceremony replication; documented failover runbook and RTO/RPO |
| Credential-stuffing / DoS against AuthN | Elevated 401/403 rate; risk of HSM saturation | Per-identity rate-limit alerts | Rate limits ahead of the HSM tier; circuit breaker sheds load before it reaches the crypto core |
| Ciphertext/wrapped-DEK corruption | Object unreadable even with valid access | Checksum mismatch on read | Checksum stored alongside ciphertext; cross-AZ replication; treated as a data-loss incident requiring restore |

---

## 16. Scalability & Sharding

### Shard by `vault_id`, Not by Request Hash

Both the metadata store and the HSM cluster are sharded by tenant (`vault_id`), never by an arbitrary hash of the request. This is the same *locality-over-even-distribution* principle as domain-based sharding in a crawler frontier, but for a different reason: a crawler shards by domain to preserve **politeness locality**; a key vault shards by tenant to preserve **key locality** (a tenant's KEK must never be split across HSM partitions) and to **bound blast radius** — compromise of one HSM partition affects only the tenants pinned to it, never the whole platform.

### Data Plane vs. Management Plane

These scale independently. Data-plane traffic (get/use secrets and keys) runs orders of magnitude higher QPS than management-plane traffic (create/configure vaults) and is provisioned and rate-limited separately so a management-plane spike (e.g., a bulk provisioning script) can't starve production secret reads.

### What Scales Freely vs. What Doesn't

Non-sensitive metadata reads scale horizontally via ordinary read replicas. **Access-policy reads deliberately do not** — naive read replicas reintroduce the read-after-revoke problem from Section 7. They scale instead via a fast, strongly consistent path (e.g., a Raft-replicated policy store with an aggressively short-TTL cache in front of it).

---

## 17. Multi-Region Replication & Key Ceremony

This is the hardest problem in a global key vault, and the section most worth spending interview time on if the interviewer pushes toward "what about disaster recovery?"

### The Problem

Replicating root/master key material across regions without ever exposing it in transit, and without letting any single machine or operator possess the complete key at any point.

### Key Ceremony Pattern

```
1. Root key generated inside the source HSM
2. Split into N shares via Shamir's Secret Sharing
3. Each share transported to a different trusted officer / target-region HSM
   via an out-of-band, witnessed process
4. Shares reconstructed only inside the destination HSM's protected memory —
   never persisted, logged, or visible outside the HSM boundary at any point
```

### Automated Alternative: Attested HSM-to-HSM Channel

Cloud-native platforms increasingly replace the manual ceremony with **mutual remote attestation** — each HSM cryptographically proves to the other that it is genuine, unmodified hardware/firmware — allowing automated secure key transport. This is faster and removes human ceremony overhead, but shifts trust onto the vendor's attestation chain and firmware integrity, a tradeoff worth naming explicitly.

### Active-Passive, Not Active-Active, for the Crypto Core

Most vault platforms run **active-passive per vault**: one authoritative region performs writes and crypto operations; ciphertext and metadata replicate asynchronously to secondary regions for read availability and failover. Active-active key state — two HSM clusters independently able to mutate the same KEK — reintroduces a **split-brain risk on the single most sensitive piece of data the platform holds.** Even in an otherwise active-active platform, the crypto core specifically should stay single-writer per tenant. This is a deliberate, defensible design choice, not a limitation to apologize for.

### Stating RTO/RPO Explicitly

A Staff-level answer names concrete targets rather than leaving them implicit: e.g., **RPO near-zero** for metadata/ciphertext (synchronous within-region, async cross-region), **RTO on the order of minutes** for full regional failover, achievable because the standby region's HSM cluster already holds replicated key shares and doesn't need a live ceremony at failover time.

---

## 18. Senior vs Staff Answer Differentiators

### Senior-level expectations

- Correctly identify the major components (AuthN, AuthZ, storage, audit)
- Explain encryption at rest and in transit
- Mention RBAC and basic audit logging
- Handle obvious failure modes (service crash, storage outage)
- Provide basic capacity math

### Staff-level expectations (additional)

| Topic | What Staff-level adds |
|---|---|
| Key hierarchy | Full envelope encryption chain (root → KEK → DEK); explains *why* the KEK never leaves the HSM boundary |
| Keys vs. Secrets | Private key material for a Key object never leaves the vault, even to its owner — crypto ops happen as RPCs into the vault |
| Threshold cryptography | Shamir's Secret Sharing for root-key unsealing; no single operator holds the complete key |
| Fail-closed / fail-open | Deliberate, reasoned split between control-plane (fail-closed) and high-volume read (fail-open, monitored) audit paths |
| CAP theorem | Explicit per-component decision — and explains *why this system inverts the typical AP-default* toward CP |
| Read-after-revoke | Proactively names the stale-ACL-replica problem and its quorum-read fix |
| Multi-region | Key ceremony / threshold sharing for cross-region root-key replication; active-passive justification for the crypto core specifically |
| Rotation | Dual-version validity window to avoid a "flag day" outage; silent-renewal-failure trap for certificates |
| Sharding | Shards by tenant for *key locality and blast-radius containment*, not for even load distribution |
| Storage design | States the "two independent compromises required" invariant explicitly as the payoff of the key hierarchy |
| Insider threat | Just-in-time access, dual control, and break-glass with mandatory heavy audit flagging |

### The single most important Staff differentiator

**Explaining *why* the security boundary holds, not just *that* it exists.** Any candidate can say "keys are encrypted at rest." A Staff engineer explains that the KEK never leaves the HSM, that compromising storage alone yields nothing decryptable, that no single operator can reconstruct the root key, and that these are three *independent* layers of defense — each one deliberately designed to fail safely on its own.

---

## 19. Interview Time Allocation

For a 45-minute system design session:

| Phase | Time | Focus |
|---|---|---|
| Requirements & scope | 5 min | Confirm secrets vs. keys vs. certs distinction; confidentiality/audit non-functionals |
| Capacity estimation | 4 min | HSM ops/sec, audit throughput, contrast with typical storage-bound systems |
| High-level architecture | 5 min | Draw all components; call out the three-gate (AuthN/AuthZ/rate-limit) fan-in |
| Cryptographic Core (deep dive) | 12 min | Key hierarchy, envelope encryption, HSM boundary, threshold unsealing |
| AuthN/AuthZ | 8 min | Identity flow, policy layers, read-after-revoke problem |
| Audit logging | 5 min | Fail-closed vs. fail-open split, tamper-evidence |
| CAP positioning | 3 min | Per-component table, explain the CP-default inversion |
| Multi-region / DR (if time) | 3 min | Key ceremony, active-passive crypto core |

**What to cut if short on time:** certificate-lifecycle detail and multi-region key ceremony depth. **Never cut:** the key hierarchy/envelope-encryption walkthrough, or the AuthZ read-after-revoke problem — these are the two places interviewers probe hardest.

---

## 20. Quick-Reference Cheatsheet

```
KEY HIERARCHY
─────────────
Root Key (HSM, non-exportable)
  → wraps Vault Master Key (KEK, per tenant)
    → wraps Data Encryption Key (DEK, per object)
      → encrypts the actual payload

KEY ALGORITHMS
──────────────
Payload encryption:  AES-256-GCM (DEK-level)
Key wrapping:        RSA-OAEP or AES-KW (KEK wraps DEK, inside HSM)
Unsealing:           Shamir's Secret Sharing — quorum (e.g. 3-of-5) reconstructs root key
Audit integrity:     Hash-chained records (Merkle-chain), WORM storage

KEY NUMBERS
───────────
100,000 vaults  →  500,000 req/sec platform peak
~75,000 ops/sec are true HSM-bound crypto (not cacheable)
~150 HSM partitions (3× headroom, N+2 redundancy)
~500 MB/s durable audit-write throughput
6.25 TB encrypted payload storage — NOT the bottleneck

KEY SYSTEMS
───────────
Metadata store:     wide-row (Cassandra-style), partition by vault_id
Access policy store: strongly consistent (Raft-replicated)
Payload store:       object storage, AP replication (ciphertext is inert)
Audit pipeline:      Kafka → WORM / immutable store
Crypto core:          HSM cluster, sharded by vault_id, active-passive per tenant

CAP DECISIONS
─────────────
CP:  AuthZ decisions, access-policy store, HSM/crypto core, version pointer,
     audit writes for mutating events
AP:  encrypted payload blobs, descriptive metadata, rate-limiter state,
     audit writes for high-volume read events

SECURITY RULES
──────────────
1. The KEK and root key never leave the HSM boundary — only wrapped
   material and short-lived DEKs cross it
2. Private key material for "Key" objects never leaves the vault, ever
3. No single operator can reconstruct the root key (threshold sharing)
4. Fail closed on AuthZ and on mutating-event audit writes
5. Never log the secret payload itself — only the operation and outcome
6. Two independent compromises required: storage alone yields nothing
   decryptable without the HSM

STAFF-LEVEL SIGNALS TO HIT
───────────────────────────
✓ Full envelope-encryption walkthrough with the "why" at each hop
✓ Keys-never-leave-the-vault distinction from Secrets
✓ Threshold cryptography for root-key unsealing
✓ Explicit fail-closed vs. fail-open split, reasoned not blanket
✓ CAP table that explains WHY this system inverts the typical AP default
✓ Read-after-revoke problem named proactively, with a quorum-read fix
✓ Active-passive justification for the crypto core specifically
✓ Sharding framed as key-locality/blast-radius, not load distribution
✓ Concrete RTO/RPO targets for regional failover
```

---

## 21. Related Guides

- **CAP/PACELC Theorem** — full framework behind the per-component CAP table in Section 14.
- **Distributed Key-Value Store (Dynamo-style)** — quorum reads/writes and staleness handling referenced in the rotation cache-invalidation discussion (Section 10).
- **Leader Election in Distributed Systems** — underlies the Raft-replicated access-policy store used to solve the read-after-revoke problem (Section 7).
- **Circuit Breaker Pattern** — the SDK-side resilience mechanism referenced in Section 5 to keep client retries from overwhelming a degraded vault.
- **Sharding Strategies** — general sharding tradeoffs; this guide's tenant-based sharding (Section 16) is a specific instance driven by key locality rather than load distribution.

---

*Guide compiled for Senior & Staff-level system design interview preparation.*
*Topics: Key Management, Envelope Encryption, HSM, Access Control, Audit Logging, CAP Theorem.*
