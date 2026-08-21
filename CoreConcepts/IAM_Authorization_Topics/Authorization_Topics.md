# Security Fundamentals: AuthN, AuthZ & IAM

## Table of Contents

1. [Authentication vs. Authorization](#1-authentication-vs-authorization)
   - 1.1 [Authentication (AuthN)](#11-authentication-authn--who-are-you)
   - 1.2 [Authorization (AuthZ)](#12-authorization-authz--what-are-you-allowed-to-do)
   - 1.3 [Key Distinction](#13-key-distinction)
   - 1.4 [A Concrete Flow](#14-a-concrete-flow)
   - 1.5 [Mnemonic](#15-mnemonic)
   - 1.6 [A Common Pitfall](#16-a-common-pitfall)
2. [IAM Fundamentals](#2-iam-fundamentals)
   - 2.1 [Core Components](#21-core-components)
   - 2.2 [Common Access Control Models](#22-common-access-control-models)
     - 2.2.1 [RBAC — Role-Based Access Control](#221-rbac--role-based-access-control)
     - 2.2.2 [ABAC — Attribute-Based Access Control](#222-abac--attribute-based-access-control)
     - 2.2.3 [PBAC — Policy-Based Access Control](#223-pbac--policy-based-access-control)
     - 2.2.4 [DAC — Discretionary Access Control](#224-dac--discretionary-access-control)
   - 2.3 [Why It Matters](#23-why-it-matters)
3. [Protocols & Tokens](#3-protocols--tokens)
   - 3.1 [OAuth 2.0](#31-oauth-20)
   - 3.2 [OpenID Connect (OIDC)](#32-openid-connect-oidc)
   - 3.3 [JWT — JSON Web Token](#33-jwt--json-web-token)

---

## 1. Authentication vs. Authorization

These two terms get mixed up constantly, but they answer different questions.

### 1.1 Authentication (AuthN) — *"Who are you?"*

Verifying identity. You prove you are who you claim to be.

**Examples:**
- Entering a password
- Using a fingerprint or Face ID
- A one-time code from an authenticator app
- A signed JWT proving you're a valid user

### 1.2 Authorization (AuthZ) — *"What are you allowed to do?"*

Determining permissions. Once your identity is confirmed, authorization decides what resources you can access and what actions you can take.

**Examples:**
- An admin can delete users; a regular user can only view their own profile
- A role-based access control (RBAC) policy gating an API endpoint

### 1.3 Key Distinction

| | Authentication | Authorization |
|---|---|---|
| Question | Who are you? | What can you do? |
| Happens | First | After authentication |
| Data used | Credentials (password, biometrics, tokens) | Roles, permissions, policies |
| Example failure | 401 Unauthorized | 403 Forbidden |
| Common standards | OAuth 2.0 (partially), OpenID Connect, SAML | RBAC, ABAC, OAuth 2.0 scopes |

### 1.4 A Concrete Flow

1. You log into a system with a username/password → **authentication** happens, the system confirms you're "Alice."
2. You try to access `/admin/settings` → **authorization** kicks in, checking whether Alice's role permits that action.

### 1.5 Mnemonic

AuthN comes before AuthZ — alphabetically, and in the sequence of events. You have to know *who* someone is before you can decide *what* they're allowed to do.

### 1.6 A Common Pitfall

OAuth 2.0 is often miscast as an "authentication protocol," but it's fundamentally an **authorization** framework — it grants scoped access to resources. OpenID Connect (OIDC) is the layer built on top of OAuth 2.0 that actually handles authentication. This distinction trips people up a lot in system design interviews and real-world API design.

---

## 2. IAM Fundamentals

**Identity and Access Management (IAM)** is the umbrella discipline that authentication and authorization both live under. It's the set of policies, processes, and technologies an organization uses to make sure the right identities (people, services, devices) have the right access to the right resources, at the right time, for the right reasons.

### 2.1 Core Components

- **Identity lifecycle management** — creating, updating, and deactivating accounts as people join, change roles, or leave (often called *provisioning* and *deprovisioning*).
- **Authentication** — verifying identity (see Section 1).
- **Authorization** — granting permissions (see Section 1).
- **Directory services** — the system of record for identities, e.g. Active Directory, LDAP, or a cloud directory like Azure AD / Entra ID or AWS IAM Identity Center.
- **Single Sign-On (SSO)** — one login session grants access across multiple applications.
- **Multi-Factor Authentication (MFA)** — requiring more than one proof of identity (e.g. password + authenticator app).
- **Federation** — trusting identities issued by another system, typically via SAML, OAuth 2.0, or OpenID Connect.
- **Privileged Access Management (PAM)** — extra controls (approval workflows, session recording, just-in-time access) around high-risk accounts like admins.
- **Audit & compliance** — logging who accessed what, and proving it for standards like SOC 2, HIPAA, or ISO 27001.

### 2.2 Common Access Control Models

| Model | How access is decided |
|---|---|
| **RBAC** (Role-Based) | Access tied to a role (e.g. "Editor," "Admin") |
| **ABAC** (Attribute-Based) | Access tied to attributes (department, location, time of day) |
| **PBAC** (Policy-Based) | Access tied to centrally defined policies/rules |
| **DAC** (Discretionary) | Resource owner decides who gets access |

#### 2.2.1 RBAC — Role-Based Access Control

Access is granted based on the **role** a user is assigned, rather than to the individual directly. Each role bundles a fixed set of permissions, and users inherit whatever permissions their role carries.

- **How it works:** An admin defines roles like `Viewer`, `Editor`, or `Admin`, each with a preset permission set. Users are then assigned one or more roles.
- **Example:** In a company wiki, everyone in the `Editor` role can create and edit pages, but only `Admin` can delete the wiki entirely.
- **Strengths:** Simple to reason about, easy to audit ("who has the Admin role?"), scales well when job functions map cleanly to permission sets.
- **Weaknesses:** Can lead to *role explosion* — as exceptions pile up, teams create ever more granular roles (`Editor-NoDelete`, `Editor-EMEA`, etc.) to cover edge cases.
- **Best for:** Organizations with clearly defined job functions and relatively stable permission needs (most enterprise SaaS apps use RBAC as a baseline).

#### 2.2.2 ABAC — Attribute-Based Access Control

Access is decided dynamically based on **attributes** of the user, the resource, and the environment — evaluated against rules at request time, rather than a fixed role.

- **How it works:** A policy engine evaluates conditions like `user.department == resource.department AND time.hour BETWEEN 9-17`.
- **Example:** A finance employee can view invoices only for their own region, and only during business hours.
- **Strengths:** Extremely fine-grained and flexible; can express context-aware rules that RBAC can't (time of day, location, device trust level).
- **Weaknesses:** Harder to audit ("who can access X?" requires evaluating rules, not just reading a role list); more complex to design and debug.
- **Best for:** Environments needing fine-grained, contextual control — e.g. healthcare, finance, or multi-tenant SaaS platforms.

#### 2.2.3 PBAC — Policy-Based Access Control

Access is governed by **centrally defined, human-readable policies** that combine roles, attributes, and business rules into one place — often thought of as a superset or formalization of RBAC + ABAC.

- **How it works:** Policies are written declaratively (e.g. using a language like Open Policy Agent's Rego, or AWS IAM policy JSON) and evaluated by a central policy engine whenever an access decision is needed.
- **Example:** A policy states "Users with role `Manager` can approve expense reports under $5,000 for their own direct reports only" — combining role, attribute, and business logic in a single rule.
- **Strengths:** Centralizes governance so policies can be versioned, tested, and audited like code; decouples policy logic from application code.
- **Weaknesses:** Requires investment in a policy engine and the discipline to maintain policies as the organization evolves.
- **Best for:** Larger organizations that want a single, auditable source of truth for access rules across many systems.

#### 2.2.4 DAC — Discretionary Access Control

Access is left to the **discretion of the resource owner** — whoever creates or owns a resource decides who else can access it.

- **How it works:** The owner of a file, folder, or document grants or revokes access directly, without needing a central admin.
- **Example:** Sharing a Google Doc with specific people, or setting a file's permissions on a shared drive.
- **Strengths:** Simple and flexible for end users; no central bottleneck for everyday sharing decisions.
- **Weaknesses:** Hard to govern at scale — permissions sprawl as individual owners make inconsistent decisions, and there's no guarantee of least-privilege enforcement.
- **Best for:** Collaborative, user-driven tools (file sharing, documents) rather than systems handling sensitive or regulated data.

### 2.3 Why It Matters

IAM is often called the "first line of defense" in security — most breaches trace back to compromised credentials or excessive access rather than exotic exploits. A mature IAM program follows the **principle of least privilege**: every identity gets only the access it needs, nothing more.

---

## 3. Protocols & Tokens

Section 1 and 2 covered the *concepts* of authentication, authorization, and IAM. This section covers the concrete **protocols and token formats** that implement those concepts in real systems.

### 3.1 OAuth 2.0

OAuth 2.0 is an **authorization framework** — it lets one application get limited access to a user's resources on another service, *without* ever seeing the user's password.

- **What problem it solves:** Before OAuth, apps often asked users to hand over their actual username/password for a third-party service (the "password anti-pattern"). OAuth replaces that with scoped, revocable access tokens instead.
- **Core roles:**
  - **Resource Owner** — the user who owns the data.
  - **Client** — the app requesting access (e.g. a photo-printing service).
  - **Authorization Server** — issues tokens after the user consents (e.g. Google's auth server).
  - **Resource Server** — hosts the protected data and accepts the token (e.g. Google Photos API).
- **Example flow:** You click "Sign in with Google" on a photo-printing site. Google asks "Allow this app to view your photos?" You approve, and the printing site receives an **access token** scoped to `photos.readonly` — it never sees your Google password.
- **Key artifact:** The **access token** — a credential the client presents to the resource server on every request, usually short-lived and often a JWT (see 3.3).
- **Common grant types:** Authorization Code (most common, used by web/mobile apps), Client Credentials (machine-to-machine), Refresh Token (renewing access without re-login).
- **Important nuance:** OAuth 2.0 by itself only answers "what can this app access?" — it does **not** tell the app who the user *is*. That gap is exactly what OIDC fills (see 3.2).

### 3.2 OpenID Connect (OIDC)

OpenID Connect is an **authentication layer built directly on top of OAuth 2.0**. It takes OAuth's authorization machinery and adds a standardized way to verify user identity.

- **What problem it solves:** OAuth 2.0 has no built-in concept of "login" or "identity" — everyone rolled their own conventions, leading to inconsistent and often insecure implementations. OIDC standardizes this.
- **How it extends OAuth:** OIDC adds one new token — the **ID Token** — alongside OAuth's access token. The ID Token is always a JWT (see 3.3) and contains verified facts about the user.
- **Example flow:** Same "Sign in with Google" click as before, but now Google returns both an access token (for API calls) *and* an ID token containing claims like `sub` (user ID), `email`, `name`, and `email_verified`. The app decodes the ID token to know exactly who logged in.
- **Key artifact:** The **ID Token** (JWT) — proof of authentication, meant to be consumed by the client app itself, not sent to other APIs.
- **Relationship to OAuth 2.0:** Think of OAuth 2.0 as the transport/authorization plumbing, and OIDC as the identity layer riding on top of it. This is why "Sign in with Google/Microsoft/Apple" buttons are OIDC in practice, even though people casually call it "OAuth login."

### 3.3 JWT — JSON Web Token

A JWT is a **compact, self-contained token format** used to represent claims (statements) about a user or entity, digitally signed so the receiver can verify it hasn't been tampered with.

- **Structure:** Three Base64URL-encoded parts separated by dots — `header.payload.signature`.
  - **Header** — specifies the token type and signing algorithm (e.g. `HS256`, `RS256`).
  - **Payload** — the **claims**, e.g. `sub` (subject/user ID), `iss` (issuer), `exp` (expiration time), plus any custom claims like `role`.
  - **Signature** — created by signing the header + payload with a secret or private key, so any tampering invalidates it.
- **Example:** A decoded JWT payload might look like:
  ```json
  { "sub": "1234567890", "name": "Alice", "role": "admin", "exp": 1735689600 }
  ```
- **Why it matters:** JWTs are **self-contained** — a server can verify the signature and trust the claims inside without a database lookup or a call back to the issuing server. This makes them fast and easy to scale in distributed systems.
- **Where it shows up:** Used as the OAuth 2.0 **access token** format, and *always* as the OIDC **ID token** format. Also common as a general-purpose session/API token outside OAuth/OIDC entirely.
- **Important caveat:** JWTs are signed, not encrypted, by default — anyone can decode and read the payload (it's just Base64, not secret). Never put sensitive data like passwords in a JWT payload. Also, because they're self-contained, revoking a JWT *before* it expires is hard — this is usually solved with short expiration times plus refresh tokens.
