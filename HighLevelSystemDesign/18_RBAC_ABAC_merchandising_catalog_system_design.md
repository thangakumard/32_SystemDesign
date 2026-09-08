# RBAC + ABAC Authorization for a Merchandising Catalog Platform (Azure)

## Table of Contents
1. Problem Statement & Requirements
2. Domain Model: What We're Protecting
3. Why RBAC Alone Isn't Enough (and Why Pure ABAC Isn't Either)
4. Hybrid Authorization Model — NIST ABAC (PEP / PDP / PIP / PAP)
5. Architecture on Azure
6. Request Walkthrough (End-to-End)
7. CAP/PACELC Positioning
8. Failure Modes & Mitigations
9. Scalability & Partitioning
10. Senior vs. Staff Differentiators
11. 45-Minute Interview Time Allocation
12. Quick-Reference Cheatsheet
13. Interview Delivery Notes (Audience: Microsoft Security Team)
14. Related Guides

---

## 1. Problem Statement & Requirements

**System:** a merchandising catalog platform holding product catalog data, taxonomy (category tree), pricing (retail + cost/margin), and inventory (per-location stock). Internal users only — merchandisers, category managers, pricing analysts, inventory managers, finance, regional managers, read-only reporting/BI consumers.

**Functional requirements**
- Different user populations need different *operations*: browse/search catalog, create/reorganize taxonomy, view/edit retail price, view/edit cost price, adjust inventory counts, run reports.
- Access must be scoped not just by operation but by *which records*: a Category Manager for Electronics shouldn't touch Apparel; a Pricing Analyst for EMEA shouldn't edit US prices.

**Non-functional / security requirements (state these explicitly — this is what a security team is listening for)**
- **Deny-by-default, least privilege.**
- **Data classification drives access, not just role**: cost price / margin is confidential-financial; retail price and taxonomy are internal/public-facing; inventory counts are internal.
- **Data residency / compliance**: EU pricing and inventory data subject to GDPR — access constrained by region, not just role.
- **Auditability**: every allow/deny decision logged, tamper-evident, retained for compliance (SOC2 / ISO 27001).
- **Fast revocation**: a terminated employee or revoked role must lose access in minutes, not at token expiry.
- **Zero Trust posture**: never treat network location as a security boundary — every request authenticated and authorized independently.

---

## 2. Domain Model: What We're Protecting

**Resources**
- `Product` (SKU) — belongs to a `Category`, has `brand`, `department`.
- `Category` (taxonomy node) — hierarchical, owned by a department/category manager.
- `PriceRecord` — `{retailPrice, costPrice, currency, region, effectiveDate, classification}`. `costPrice` is tagged `confidential-margin`.
- `InventoryRecord` — `{sku, locationId, quantity, reserved, region}`.

**Subject (user) attributes** — sourced from HR/Entra: `department`, `region`, `clearance` (e.g. `finance-cleared`), `employmentStatus`, device-compliance/session-risk (from Conditional Access).

**Environment attributes** — request time, source network (corp vs. public), geo-location, session risk score.

**Resource attributes** — denormalized onto the record itself: `ownerDepartment`, `region`, `classification`, `brand/tenant`.

This is the ABAC "policy input" — subject × resource × environment attributes, evaluated against an action.

---

## 3. Why RBAC Alone Isn't Enough (and Why Pure ABAC Isn't Either)

A pure RBAC model gives `PricingAnalyst` → `read/write PriceRecord`. That's too coarse: it can't express "only for your own region" or "only if finance-cleared, regardless of role." A pure ABAC model *can* express everything, but pushes every single permission — including "can this user even attempt to open the pricing screen" — into policy evaluation, which is harder to provision (HR-driven joiner/mover/leaver processes assign *roles*, not attribute bundles), harder to reason about at a glance, and harder to audit ("show me everyone who can edit prices" becomes a policy-simulation query instead of a role-membership query).

**The Staff-level answer is a hybrid, layered model**: RBAC narrows the *action space* (what operations are even possible for this identity), ABAC narrows the *resource-instance space* within that action (which specific rows this identity can act on). They compose — RBAC is not replaced by ABAC, it's the coarse first filter.

---

## 4. Hybrid Authorization Model — NIST ABAC (PEP / PDP / PIP / PAP)

Use the NIST 800-162 decomposition — naming it explicitly is itself a signal you've studied real authorization architecture, not just "if user.role == admin":

- **PEP (Policy Enforcement Point)** — intercepts the request and enforces the decision. Two PEPs in this design: the API gateway (coarse) and the service itself (fine-grained).
- **PDP (Policy Decision Point)** — evaluates policy against subject/resource/environment attributes, returns allow/deny (+ reason).
- **PIP (Policy Information Point)** — supplies the attributes the PDP needs, from wherever they live (identity provider, database).
- **PAP (Policy Administration Point)** — where policies are authored, versioned, and published.

Splitting these out (rather than hardcoding checks in each service) means policy changes without a code deploy, a single place security reviews policy, and a single format for audit.

---

## 5. Architecture on Azure

```
                                   ┌─────────────────────────┐
                                   │   Microsoft Entra ID     │
                                   │ App Roles (RBAC defs),   │
                                   │ Security Groups, PIM,    │
                                   │ Custom Security Attrs,   │
                                   │ Continuous Access Eval.  │
                                   └────────────┬─────────────┘
                                                │ OIDC/OAuth2 → JWT (roles claim)
                                                ▼
Client ──HTTPS──▶ Azure Front Door + WAF ──▶ Azure API Management  (PEP-1, coarse)
                                                │  • validate JWT sig / exp / aud
                                                │  • coarse RBAC: role→API scope check
                                                │  • rate limit, mTLS to backend
                                                ▼
                              ┌───────────────────────────────────┐
                              │   Catalog / Price / Inventory      │
                              │   services (AKS) — PEP-2, fine     │
                              └───────┬───────────────┬────────────┘
                          decision    │               │ attribute lookup
                                      ▼               ▼
                         ┌────────────────────┐  ┌────────────────────────┐
                         │  PDP: OPA sidecar    │◀─│ PIP: subject attrs      │
                         │  (Rego/Cedar policy) │  │ (Entra Custom Security  │
                         │  local, sub-ms decide│  │ Attrs, short-TTL cache) │
                         └─────────┬────────────┘  │ resource attrs (from    │
                                   │                │ the record itself)     │
                              signed bundle          └────────────────────────┘
                                   │
                         ┌─────────▼────────────┐
                         │  PAP: Git repo policy  │
                         │  as code + CI (opa test│
                         │  ) → signed bundle in  │
                         │  Blob Storage/CDN      │
                         └────────────────────────┘

 Data plane (resource attrs live with the data — defense-in-depth, second gate):
  Catalog + Taxonomy → Cosmos DB, partition = tenant/categoryRoot
  Price (incl. cost)  → Azure SQL DB, Row-Level Security (SESSION_CONTEXT) +
                          Always Encrypted / dynamic data masking on costPrice
  Inventory            → Cosmos DB / Redis, partition = locationId

 Cross-cutting:
  Every PDP decision → Event Hub → Log Analytics / Sentinel (immutable audit)
  Service-to-service calls → Managed Identity (no shared secrets), OBO token
  flow to preserve the original user's identity across hops
```

**Why each piece, briefly:**
- **Entra ID App Roles + Security Groups** = the RBAC layer. Assign roles to groups, not individual users, to avoid assignment sprawl and to piggyback on existing HR-driven group membership.
- **Entra ID PIM** — sensitive roles (e.g., cost-price edit) are activated just-in-time and time-boxed rather than standing — reduces blast radius of a compromised account.
- **Entra ID Custom Security Attributes** — the natural place to hold ABAC subject attributes (department, region, clearance); they're queryable via Graph and are the same mechanism Azure's own attribute-based access control features build on.
- **Continuous Access Evaluation (CAE)** — subscribes to critical events (account disabled, password changed, admin revoke, elevated risk) so a still-unexpired token can be rejected within minutes. This is the direct answer to "how fast can you revoke access" — a question a security team will ask.
- **API Management as PEP-1** — cheap to reject clearly-unauthorized calls before they reach business logic; keeps attribute-heavy fine-grained logic out of the gateway (the gateway shouldn't need a database round-trip).
- **OPA sidecar as PEP-2/PDP** — co-located with the service, decision is local (no extra network hop on the hot path), policy bundle is pulled/verified periodically from the PAP — the same "avoid a synchronous hard dependency" instinct as a circuit breaker.
- **Azure SQL Row-Level Security** for price data — a `SESSION_CONTEXT`-driven filter predicate (SELECT) and block predicate (INSERT/UPDATE) enforced *inside the database engine*, independent of application code. This is the defense-in-depth layer: even a buggy app-layer ABAC check can't return or write out-of-scope rows. `Always Encrypted`/dynamic data masking on the `costPrice` column adds a third gate specifically for the confidential-margin classification.
- **Cosmos DB partitioning** for catalog/taxonomy/inventory — partition key doubles as an isolation boundary (tenant/category/location), and Cosmos resource tokens can scope access to a partition as an analogous defense-in-depth control (Cosmos has no native row-level security).
- **Event Hub → Log Analytics/Sentinel** — audit writes are async (don't block the authorization decision) but durable; Sentinel can flag anomalies like a burst of denies from one identity.

---

## 6. Request Walkthrough (End-to-End)

*A Pricing Analyst based in EMEA requests the cost price of a US SKU.*

1. Client sends the Entra-issued JWT to Front Door → API Management.
2. APIM validates the token and confirms role `PricingAnalyst` is permitted to call `GET /price` at all (coarse RBAC) → forwards.
3. The Price service (PEP-2) calls the local OPA sidecar with `{subject: oid, action: read.costPrice, resource: sku:123}`; OPA fetches resource attributes for `sku:123` (`region=US, classification=confidential-margin`) and subject attributes (`region=EMEA, clearance=[]`) from the PIP.
4. Policy: `subject.region == resource.region AND "finance" in subject.clearance`. EMEA ≠ US → **deny**, even though the role check passed.
5. Deny (with reason) is emitted to Event Hub; the API returns 403 with no leakage of the actual value.
6. Had it been an allow, the SQL query still runs with `SESSION_CONTEXT` set — RLS re-checks region as a second, independent gate, and the `costPrice` column only decrypts for callers whose key access matches the confidential-margin classification.

That last step is the sentence to say out loud in the interview: **two independent enforcement points reach the same conclusion by different mechanisms** — that's what "defense in depth" concretely looks like, not just a buzzword.

---

## 7. CAP/PACELC Positioning

| Component | Position | Why |
|---|---|---|
| Token validation (APIM) | AP / EL | Stateless signature check against cached JWKS keys — no cross-node consistency needed. |
| PDP decision (OPA sidecar) | AP / EL for low-sensitivity ops; **fail-closed** for confidential/financial actions | Local bundle favors latency; a short staleness window is fine for public catalog reads, unacceptable for cost-price/finance actions. |
| Subject attribute store (PIP) | CP-leaning for privilege-affecting attributes | A stale "finance-cleared" attribute after revocation is a security incident, not just an inconsistency — short TTL + event-driven invalidation over pure availability. |
| Price data (Azure SQL) | CP | Financial correctness requires the RLS predicate to see committed state, not stale replicas. |
| Inventory counts | AP/EL for display, stronger guarantee on the decrement path | Eventual consistency fine for "in stock" display; reservation/decrement needs optimistic concurrency + idempotency keys. |
| Audit log write path | AP for the hot path (never block an auth decision on it) but durability is non-negotiable | At-least-once delivery with retry; compliance needs no lost records, not read-your-write. |

---

## 8. Failure Modes & Mitigations

| Failure | Impact | Mitigation |
|---|---|---|
| PDP/OPA sidecar unreachable | No decision available | Fail-closed by default for confidential/financial resources; fail-open only for explicitly whitelisted low-risk public reads, with the fallback itself logged. Circuit breaker around PDP calls. |
| Token still valid but access was just revoked | Stale privilege persists until token expiry | Short-lived access tokens + Continuous Access Evaluation for near-real-time revocation. |
| Subject attribute staleness (department/region changed mid-day) | Wrong-scope access | Short-TTL PIP cache + event-driven invalidation from the Entra/HR sync. |
| Policy bug ships a bad rule | Mass over- or under-permissioning | Policy-as-code with `opa test` unit tests in CI, canary rollout of new bundles to a subset of pods, signed & versioned bundles, fast rollback. |
| Confused deputy — service A forwards its own broad identity instead of the caller's | Privilege escalation | On-behalf-of (OBO) token flow — propagate the *user's* claims through the call chain for user-scoped operations, not the service's Managed Identity permissions. |
| Cross-tenant/region data bleed in a multi-brand catalog | Compliance violation (GDPR) | Partition-key isolation in Cosmos DB **and** RLS in SQL as an independent second layer; resource tokens scoped per partition. |
| Audit pipeline backpressure/outage | Loss of compliance trail | Local buffering + retry with idempotency keys before ack; treat the audit write as at-least-once critical even though it's async. |
| Catch-all "Admin" role used as a workaround | Privilege creep over time | Entra ID Access Reviews with mandatory periodic recertification; PIM for anything beyond baseline read access. |

---

## 9. Scalability & Partitioning

- **Catalog/Taxonomy (Cosmos DB)**: partition by tenant/category root. A hot category during a promo event is the same hot-partition class of problem as celebrity fan-out — mitigate with a synthetic partition key (categoryRoot + hash suffix) rather than one giant partition.
- **Policy distribution**: OPA sidecars pull a compiled, signed bundle from Blob Storage behind a CDN — the decision path is entirely local, so the PDP scales for free with the service mesh. The actual bottleneck moves to the PAP build pipeline, which is low-QPS (only fires on policy change), so it's not a hot-path concern.
- **PIP (attribute store)**: read-heavy — cache subject attributes in Redis close to each service; bound staleness with a short TTL (~5 min) plus event-driven invalidation on attribute change.
- **Audit log**: partition Event Hub by date/entity-type; tier retention in Log Analytics (hot ~30 days, cold/archive for the multi-year compliance window) to separate query performance from retention cost.
- **Regional/data residency**: deploy an EU region "stamp" with its own Cosmos DB, SQL DB, and attribute store. ABAC policy — not network topology — is what actually enforces the residency boundary; don't rely on "this call arrived via the EU load balancer" as a security control (that's a Zero Trust violation in miniature).

---

## 10. Senior vs. Staff Differentiators

**Senior-level answer:** "Add a roles table, check `role == 'PricingAdmin'` before allowing a price edit, put the role in the JWT."

**Staff-level answer proactively:**
- Names the NIST PEP/PDP/PIP/PAP decomposition and explains *why* the split matters operationally (policy changes without a redeploy, one review surface for security).
- Treats RBAC and ABAC as **composable layers**, not a choice between them.
- Calls out defense-in-depth explicitly — app-layer ABAC *and* DB-layer RLS/partition isolation, so no single bug class equals a breach.
- Picks fail-open vs. fail-closed **per data classification**, not as one global default.
- Flags token/attribute staleness as a first-class correctness problem and reaches for CAE + short-TTL attribute caching, not "just check the JWT."
- Treats the audit trail as a system requirement with its own durability contract, not an afterthought log line.
- Uses OBO token flow / Managed Identity to close the confused-deputy gap in service-to-service calls.
- Ties partitioning and regional design to compliance (data residency), not only throughput.

---

## 11. 45-Minute Interview Time Allocation

- **5 min** — requirements, and explicitly classify the data by sensitivity (this framing pays off for the rest of the interview).
- **10 min** — domain model: entities, subject/resource/environment attributes.
- **10 min** — high-level architecture: Entra ID, PEP/PDP/PIP/PAP, data layer.
- **10 min** — deep dive: one concrete request walkthrough showing the layered (app + DB) enforcement.
- **5 min** — failure modes: fail-open/closed tradeoff, the revocation story (CAE).
- **5 min** — scalability/regional residency and wrap-up.

---

## 12. Quick-Reference Cheatsheet

| Concept | Azure Building Block |
|---|---|
| IdP / RBAC role definitions | Microsoft Entra ID App Roles + Security Groups |
| Just-in-time elevation | Entra ID Privileged Identity Management (PIM) |
| Near-real-time revocation | Continuous Access Evaluation (CAE) |
| Subject ABAC attributes | Entra ID Custom Security Attributes |
| Gateway-level PEP | Azure API Management |
| Fine-grained PEP/PDP | OPA (or Cedar) sidecar in AKS |
| Policy-as-code / PAP | Git repo + CI (`opa test`) → signed bundle in Blob Storage |
| Catalog/Taxonomy store | Cosmos DB, partitioned by tenant/categoryRoot |
| Price store + native DB-level ABAC control | Azure SQL DB — Row-Level Security (`SESSION_CONTEXT`), Always Encrypted on `costPrice` |
| Inventory store | Cosmos DB / Redis, partitioned by location |
| Secrets/keys | Azure Key Vault + Managed Identity |
| Audit trail | Event Hub → Log Analytics / Microsoft Sentinel, immutable storage |
| Perimeter | Azure Front Door + WAF |

---

## 13. Interview Delivery Notes (Audience: Microsoft Security Team)

- **Open with the classification, not the architecture.** A security interviewer wants to hear "I'd first classify what I'm protecting — cost price is confidential-financial, catalog is internal-facing, inventory is internal" before any component names show up. That ordering itself signals security-first thinking.
- **Say "Zero Trust" and "least privilege" only when you can immediately back them with a mechanism** (CAE, RLS block predicates, deny-by-default PDP) — naming the principle and then the concrete Azure control that implements it lands far better than the term alone.
- **Volunteer the revocation story unprompted.** "How fast can you revoke access" is one of the most common security-round follow-ups; having Continuous Access Evaluation ready before they ask is a strong signal.
- **Lead with the hybrid framing** (RBAC narrows actions, ABAC narrows instances) rather than presenting RBAC and ABAC as competing choices — this is the single idea that most separates a Senior from a Staff answer here.
- **Use their own product vocabulary correctly** (Entra ID, not "Azure AD"; App Roles vs. Custom Security Attributes as distinct mechanisms) — precision here reads as genuine hands-on familiarity rather than a keyword list.
- **Stay vague on anything you haven't personally verified.** If asked for exact throughput numbers or default token TTLs, flag them as illustrative rather than asserting a specific figure under pressure.

---

## 14. Related Guides
- **Sharding Strategies** — hot-partition patterns referenced in §9.
- **CAP/PACELC Theorem** — the per-component positioning method used in §7.
- **Circuit Breaker Pattern** — the fail-open/fail-closed and fallback reasoning used for PDP unavailability in §8.
- **Distributed Key-Value Store (Dynamo-style)** — partition/resource-token isolation analogy used for Cosmos DB.
- **Notification System** — the async, durable event-pipeline pattern reused here for the audit trail.
