# Security Fundamentals: AuthN, AuthZ & IAM

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

### 2.3 Why It Matters

IAM is often called the "first line of defense" in security — most breaches trace back to compromised credentials or excessive access rather than exotic exploits. A mature IAM program follows the **principle of least privilege**: every identity gets only the access it needs, nothing more.
