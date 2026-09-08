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
4. [Putting It Together: PhotoPrint App Example](#4-putting-it-together-photoprint-app-example)
   - 4.1 [Scenario](#41-scenario)
   - 4.2 [Step-by-Step Flow with REST Requests & Responses](#42-step-by-step-flow-with-rest-requests--responses)
   - 4.3 [How MITM Attacks Are Prevented](#43-how-mitm-attacks-are-prevented)

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
- **Key artifact:** The **access token** — a credential the client presents to the resource server on every request, usually short-lived and often a JWT (see 3.3).
- **Common grant types:** Authorization Code (most common, used by web/mobile apps), Client Credentials (machine-to-machine), Refresh Token (renewing access without re-login).
- **Important nuance:** OAuth 2.0 by itself only answers "what can this app access?" — it does **not** tell the app who the user *is*. That gap is exactly what OIDC fills (see 3.2).

#### Example Flow, Step by Step ("Connect Google Photos")

A photo-printing site wants to fetch a user's photos from Google Photos — without ever seeing the user's Google password. This uses the **Authorization Code** grant, the most common OAuth 2.0 flow.

1. **User clicks "Connect Google Photos"** on the photo-printing site (the *Client*).
2. **Client redirects the browser** to Google's authorization endpoint (the *Authorization Server*) with:
   - `response_type=code`
   - `client_id=<the app's registered ID>`
   - `redirect_uri=<where Google sends the user back>`
   - `scope=photos.readonly` — note: **no `openid` scope** here. This is what keeps it plain OAuth 2.0 rather than OIDC.
   - `state=<random value>` — CSRF protection, so the client can confirm the callback corresponds to a request it actually made.
3. **User authenticates** with Google (password, MFA, etc.) if not already signed in.
4. **User sees a consent screen** scoped strictly to data access: *"Allow this app to view your Google Photos?"* — notably, not phrased around identity or login.
5. **User approves.** Google redirects back to the client's `redirect_uri` with a short-lived **authorization code** and the original `state` value, which the client verifies matches.
6. **Client's backend exchanges the code for a token** — a server-to-server call to Google's token endpoint, authenticating with its `client_id` + `client_secret`.
7. **Google's token endpoint responds with:**
   - `access_token` — scoped to `photos.readonly`, used to call the Google Photos API.
   - `refresh_token` — optional, to get a new access token later without the user logging in again.
   - **No `id_token`.** Plain OAuth 2.0 has no concept of an identity token — that's OIDC's addition (see 3.2).
8. **Client stores the access token** and uses it on subsequent API calls, typically as an HTTP header: `Authorization: Bearer <access_token>`.
9. **Client calls the Resource Server** (Google Photos API) with the access token to fetch the user's photos. Google's API validates the token and its scope before returning any data.
10. **Access token expires** after a set period (e.g. 1 hour). When it does, the client uses the `refresh_token` to silently obtain a new `access_token` from the token endpoint, without involving the user again.

Notice what's conspicuously **absent** compared to the OIDC flow in 3.2: no `nonce`, no `id_token`, no standardized identity claims, and no UserInfo endpoint. The client walks away knowing *"I can now call the Photos API on this person's behalf,"* but it has no built-in, standardized way to know *who that person is* — it would need to call Google's proprietary profile API (outside the OAuth 2.0 spec) if it wanted that, which is precisely the gap OIDC was created to close.

### 3.2 OpenID Connect (OIDC)

OpenID Connect is an **authentication layer built directly on top of OAuth 2.0**. It takes OAuth's authorization machinery and adds a standardized way to verify user identity.

- **What problem it solves:** OAuth 2.0 has no built-in concept of "login" or "identity" — everyone rolled their own conventions, leading to inconsistent and often insecure implementations. OIDC standardizes this.
- **How it extends OAuth:** OIDC adds one new token — the **ID Token** — alongside OAuth's access token. The ID Token is always a JWT (see 3.3) and contains verified facts about the user.
- **Key artifact:** The **ID Token** (JWT) — proof of authentication, meant to be consumed by the client app itself, not sent to other APIs.
- **Relationship to OAuth 2.0:** Think of OAuth 2.0 as the transport/authorization plumbing, and OIDC as the identity layer riding on top of it. This is why "Sign in with Google/Microsoft/Apple" buttons are OIDC in practice, even though people casually call it "OAuth login."

#### Example Flow, Step by Step ("Sign in with Google")

1. **User clicks "Sign in with Google"** on the client app (e.g. a photo-printing site).
2. **Client redirects the browser** to Google's authorization endpoint, adding OIDC-specific parameters on top of the standard OAuth request:
   - `response_type=code`
   - `client_id=<the app's registered ID>`
   - `redirect_uri=<where Google sends the user back>`
   - `scope=openid email profile` — the `openid` scope is what signals *this is an OIDC request*, not plain OAuth.
   - `state=<random value>` — CSRF protection.
   - `nonce=<random value>` — OIDC-specific; binds the eventual ID token to this exact login attempt, preventing replay attacks.
3. **User authenticates** with Google (password, MFA, etc.) and sees a **consent screen**: *"This app wants to view your email address and basic profile info."*
4. **User approves.** Google redirects back to the client's `redirect_uri` with a short-lived **authorization code** and the original `state` value (which the client verifies matches).
5. **Client's backend exchanges the code for tokens** — a server-to-server call to Google's token endpoint, authenticating with its `client_id` + `client_secret`.
6. **Google's token endpoint responds with up to three tokens** — this is the key difference from plain OAuth 2.0:
   - `access_token` — for calling Google APIs (e.g. Google Photos), same as OAuth 2.0.
   - `id_token` — a JWT containing identity claims about the user. **This is new; OAuth 2.0 has no equivalent.**
   - `refresh_token` — optional, to obtain new tokens later without re-login.
7. **Client verifies the ID token** before trusting it: checks the signature against Google's published public keys (JWKS), and validates `iss` (issuer is really Google), `aud` (token was issued for this client), `exp` (not expired), and that `nonce` matches what was sent in step 2.
8. **Client decodes the ID token payload** to learn exactly who logged in, e.g.:
   ```json
   {
     "iss": "https://accounts.google.com",
     "sub": "110169484474386276334",
     "email": "alice@example.com",
     "email_verified": true,
     "name": "Alice Smith",
     "picture": "https://...",
     "aud": "client-id-of-the-app",
     "exp": 1735689600,
     "iat": 1735686000,
     "nonce": "abc123"
   }
   ```
9. **(Optional) Client calls the standardized UserInfo endpoint**, presenting the `access_token`, to fetch additional profile claims not already in the ID token (e.g. `locale`, `given_name`, `family_name`).
10. **Client establishes a session** for the user, using `sub` (a stable, unique identifier) as the durable key to look up or create the local user record — the login is now complete.

#### Does OIDC Give the Client More User Detail Than OAuth 2.0?

**Yes — this is the core reason OIDC exists.** The comparison:

| | OAuth 2.0 alone | OIDC |
|---|---|---|
| Token(s) returned | `access_token` only | `access_token` **+** `id_token` |
| Who reads the identity token | Resource server (opaque to client) | The **client itself** decodes it |
| Standardized user claims | None — up to each provider | `sub`, `email`, `name`, `email_verified`, etc., defined by spec |
| Way to fetch more profile data | Proprietary, provider-specific API | Standardized **UserInfo endpoint** |
| Replay/tampering protection for login | Not addressed | `nonce` + signature verification built in |

In pure OAuth 2.0, the client only knows it *has permission to call an API on the user's behalf* — it has no standardized way to know *who that user is*. Any "profile info" would require calling some undocumented, provider-specific endpoint. OIDC closes that gap by formalizing the ID token and the UserInfo endpoint, so identity data arrives in a predictable, verifiable shape every time — which is exactly why OIDC, not raw OAuth 2.0, is the right choice whenever "login" (not just "access") is the goal.

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

---

## 4. Putting It Together: PhotoPrint App Example

### 4.1 Scenario

**PhotoPrint App** lets a user print their photos. The user logs into PhotoPrint using their **Google account**, then browses and selects photos from **Google Photos** to print. In practice this isn't two separate flows — it's **one OIDC authorization request** that requests both the `openid` scope (for login) and the Google Photos API scope (for resource access) together, so a single token exchange returns everything needed: identity *and* API access.

### 4.2 Step-by-Step Flow with REST Requests & Responses

**Step 1 — User clicks "Sign in with Google" in PhotoPrint App.** The browser is redirected:

```http
GET /o/oauth2/v2/auth?
  client_id=photoprint-app.apps.googleusercontent.com
  &redirect_uri=https://photoprint.app/callback
  &response_type=code
  &scope=openid%20email%20profile%20https://www.googleapis.com/auth/photoslibrary.readonly
  &state=f3a9c2e1
  &nonce=n0S6Wz2Mj
  &code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM
  &code_challenge_method=S256 HTTP/1.1
Host: accounts.google.com
```

Note the combined `scope` (identity + Photos API) and the `code_challenge`/`code_challenge_method` — this is **PKCE**, central to the MITM defenses in 4.3.

**Step 2 — User authenticates and consents** at Google's own login/consent UI (not a call PhotoPrint makes directly).

**Step 3 — Google redirects back with the authorization code:**

```http
GET /callback?code=4/0AY0e-g7xY...&state=f3a9c2e1 HTTP/1.1
Host: photoprint.app
```

PhotoPrint's backend verifies `state` matches what it generated in Step 1.

**Step 4 — Client exchanges the code for tokens:**

Before this happens, the `code_verifier` sent here was already generated back in Step 1, *before* the browser was even redirected to Google. PKCE requires the verifier to exist first, because the `code_challenge` sent in Step 1 is derived from it:

1. **Generate a cryptographically random value.** The client uses a CSPRNG to create 32 random bytes (RFC 7636 recommends 32–96 bytes of entropy).
2. **Base64URL-encode it (no padding)** to produce the `code_verifier` — a 43–128 character string using only `[A-Z] [a-z] [0-9] - . _ ~`. Example: `dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk`.
3. **Store it locally** in the client (e.g. session storage, secure cookie, or server-side session) — it is never sent anywhere until this token exchange.
4. **Derive the `code_challenge`** by hashing the verifier: `code_challenge = BASE64URL-ENCODE(SHA256(code_verifier))`. This is the value that *was* sent in Step 1's `code_challenge` parameter — the verifier itself never touched the network at that point.

Example client-side generation (JavaScript):

```javascript
function base64UrlEncode(buffer) {
  return btoa(String.fromCharCode(...new Uint8Array(buffer)))
    .replace(/\+/g, '-')
    .replace(/\//g, '_')
    .replace(/=+$/, '');
}

// 1 & 2 — generate code_verifier
const randomBytes = crypto.getRandomValues(new Uint8Array(32));
const codeVerifier = base64UrlEncode(randomBytes);
// e.g. "dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk"

// 4 — derive code_challenge (sent in Step 1, not here)
const digest = await crypto.subtle.digest(
  'SHA-256',
  new TextEncoder().encode(codeVerifier)
);
const codeChallenge = base64UrlEncode(digest);
// e.g. "E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM"
```

Now, in the token request itself, the client sends the **original plaintext `code_verifier`** — not the hash — as a form field:

```http
POST /token HTTP/1.1
Host: oauth2.googleapis.com
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=4/0AY0e-g7xY...
&client_id=photoprint-app.apps.googleusercontent.com
&client_secret=GOCSPX-xxxxxxxxxxxxxxxxxxxx
&redirect_uri=https://photoprint.app/callback
&code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk
```

**How Google validates it server-side:** the token endpoint looks up the `code_challenge` it stored against this authorization `code` back in Step 1, independently computes `BASE64URL(SHA256(code_verifier))` from the verifier just received, and compares the two:

- **Match** → the request is provably coming from the same client that started the flow → tokens are issued.
- **Mismatch** → responds `400 Bad Request` with `{"error": "invalid_grant"}` and no tokens are issued.

This is precisely why a MITM who intercepts only the authorization `code` (e.g. off a malicious redirect) still can't get tokens: they have the code, but not the `code_verifier` that was generated and kept client-side back in Step 1.

Response:

```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store
Pragma: no-cache

{
  "access_token": "ya29.a0AfH6SM...",
  "expires_in": 3599,
  "refresh_token": "1//0gLd8x...",
  "scope": "openid email profile https://www.googleapis.com/auth/photoslibrary.readonly",
  "token_type": "Bearer",
  "id_token": "eyJhbGciOiJSUzI1NiIsImtpZCI6IjA5ZjI2NGI4In0.eyJpc3MiOiJhY2NvdW50cy5nb29nbGUuY29tIi..."
}
```

`Cache-Control: no-store` / `Pragma: no-cache` are mandated by spec so tokens are never cached by intermediate proxies.

**Step 5 — Client fetches Google's public signing keys and verifies the ID token's signature:**

```http
GET /oauth2/v3/certs HTTP/1.1
Host: www.googleapis.com
```

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "keys": [
    { "kid": "09f264b8", "kty": "RSA", "alg": "RS256", "use": "sig", "n": "...", "e": "AQAB" }
  ]
}
```

The client matches the `kid` in the ID token's header to a key here, verifies the RS256 signature, and checks `iss=accounts.google.com`, `aud=<its own client_id>`, `exp` not passed, and `nonce=n0S6Wz2Mj` (matches Step 1).

**Step 6 — Client decodes the ID token and establishes a session:**

```json
{
  "sub": "110169484474386276334",
  "email": "alice@example.com",
  "email_verified": true,
  "name": "Alice Smith",
  "aud": "photoprint-app.apps.googleusercontent.com",
  "iss": "accounts.google.com",
  "nonce": "n0S6Wz2Mj",
  "exp": 1735689600
}
```

Alice is now logged into PhotoPrint App, keyed on the durable `sub` identifier.

**Step 7 — User browses photos: PhotoPrint calls the Google Photos API using the *same* access token:**

```http
POST /v1/mediaItems:search HTTP/1.1
Host: photoslibrary.googleapis.com
Authorization: Bearer ya29.a0AfH6SM...
Content-Type: application/json

{ "pageSize": 25 }
```

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "mediaItems": [
    { "id": "AGj5q3...", "baseUrl": "https://lh3.googleusercontent.com/lr/...", "filename": "IMG_2024.jpg", "mimeType": "image/jpeg" }
  ]
}
```

**Step 8 — User selects photos to print; the app downloads the actual image bytes:**

```http
GET /lr/AGj5q3...=d HTTP/1.1
Host: lh3.googleusercontent.com
```

```http
HTTP/1.1 200 OK
Content-Type: image/jpeg
Content-Length: 3481200

<binary image data>
```

**Step 9 — Access token expires; client uses the refresh token to get a new one**

PhotoPrint's backend stores the `access_token` alongside an expiry timestamp it computes from `expires_in` at issuance (e.g. `issued_at + 3599s`). Two common ways this step gets triggered:

- **Proactive:** before calling the Photos API, the app checks whether the stored access token is expired (or about to expire) and refreshes first if so.
- **Reactive:** the app calls the Photos API with the token it has; Google responds `401 Unauthorized`; the app catches that, refreshes, and retries once.

Either way, the refresh call looks the same:

```http
POST /token HTTP/1.1
Host: oauth2.googleapis.com
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token
&refresh_token=1//0gLd8x...
&client_id=photoprint-app.apps.googleusercontent.com
&client_secret=GOCSPX-xxxxxxxxxxxxxxxxxxxx
```

> **Note:** `redirect_uri` and `code_verifier` are absent here — both belong strictly to the `authorization_code` grant (Step 4), where they prove the caller is the same party that initiated that specific authorization. This is a different grant type (`grant_type=refresh_token`) with no authorization code involved; legitimacy is instead proven by possessing the `refresh_token` itself plus the `client_id`/`client_secret`.

Response:

```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store
Pragma: no-cache

{
  "access_token": "ya29.a0AfH6SN2xQ...",
  "expires_in": 3599,
  "scope": "openid email profile https://www.googleapis.com/auth/photoslibrary.readonly",
  "token_type": "Bearer"
}
```

Note what's **not** in this response: no `refresh_token` and no `id_token`. Google typically only issues a `refresh_token` on the *first* authorization (unless the app forces `prompt=consent` to request a new one) — the original refresh token from Step 4 keeps working and gets reused for future refreshes too. There's also no need for a new `id_token` here; PhotoPrint's session is already established from Step 6, and this call is purely about renewing API access, not re-authenticating the user.

PhotoPrint overwrites its stored `access_token` and expiry timestamp with these new values, leaving the `refresh_token` untouched.

**Step 10 — App retries the original request with the new access token:**

```http
POST /v1/mediaItems:search HTTP/1.1
Host: photoslibrary.googleapis.com
Authorization: Bearer ya29.a0AfH6SN2xQ...
Content-Type: application/json

{ "pageSize": 25 }
```

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "mediaItems": [
    { "id": "AGj5q3...", "baseUrl": "https://lh3.googleusercontent.com/lr/...", "filename": "IMG_2024.jpg", "mimeType": "image/jpeg" }
  ]
}
```

From the user's perspective, none of this was visible — they never saw a login screen again. This refresh/retry cycle repeats silently for as long as the user keeps using PhotoPrint, until the `refresh_token` itself is revoked (e.g. the user removes PhotoPrint's access in their Google Account settings), at which point the refresh call itself fails with `{"error": "invalid_grant"}` and the app must send the user back through the full Step 1 login flow.

### 4.3 How MITM Attacks Are Prevented

No single mechanism does this alone — protection is layered across the flow:

| Layer | Mechanism | What it stops |
|---|---|---|
| **Transport** | Every request above is HTTPS/TLS, with certificate-chain validation | An on-path attacker cannot read or alter any request/response — code, tokens, headers all travel encrypted |
| **Code interception** | **PKCE** (`code_challenge` in Step 1, `code_verifier` in Step 4) | Even if an attacker captures the authorization code in transit or via a malicious app sharing the redirect URI, they can't redeem it for tokens without the original `code_verifier`, which never left the legitimate client until the TLS-protected token exchange |
| **Request integrity** | `state` parameter, echoed back and checked | Prevents an attacker from injecting their own authorization code into the victim's session (login CSRF) |
| **Token replay** | `nonce`, embedded in the ID token and checked against the original request | Stops a captured ID token from a *different* session being replayed into this one |
| **Token forgery** | ID token is a signed JWT (RS256); client verifies the signature against Google's public key (JWKS) | Even if a MITM injected a fake token, they don't hold Google's private key, so any forged token fails signature verification |
| **Token substitution** | Client checks `aud` (issued for this client) and `iss` (issued by the expected authority) | Stops a valid token issued for a *different* app from being replayed against PhotoPrint |
| **Redirect hijack** | Google only redirects the code to a pre-registered, exact-match `redirect_uri` | Blocks an attacker from registering a lookalike callback URL to steal the code |
| **Exposure window** | Access tokens are short-lived (~1 hr); refresh tokens are exchanged server-to-server only | Limits how long a stolen token (if one ever leaked) remains useful |
| **Resource server validation** | Google Photos API independently validates the token's signature/expiry/scope on every call | Rejects expired, tampered, or under-scoped tokens even if one somehow reached it |

**In short:** TLS makes the channel unreadable and untamperable in transit, PKCE makes a stolen authorization code worthless on its own, and signed tokens with `aud`/`iss`/`nonce` validation mean even a successfully intercepted token can't be forged, replayed in another context, or reused past its short lifetime.
