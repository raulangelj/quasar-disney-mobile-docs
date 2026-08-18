# Doc 16 — Identity & Auth

**Version:** v0.2.0
**Status:** Draft
**Last updated:** 2026-08-17 (STEP-1.14)
**Audience:** Mobile developers, backend team, QA

> How this React Native demo proves who someone is, what they may do, and how that swaps to a managed IdP behind our API in Phase 3 — without putting Auth0/Firebase in the app or inventing roles we do not have.

Session 1.6a is **Done**. It expands the AuthN/AuthZ stance in `architecture/06-security-threat-model.md`. Token storage (ADR-0003) and the binary navigator split are not reopened here.

## Table of Contents

1. [Authentication methods](#1-authentication-methods)
2. [Identity provider — build vs. buy](#2-identity-provider--build-vs-buy)
3. [User / account model](#3-user--account-model)
4. [Authorization model](#4-authorization-model)
5. [Multi-tenancy](#5-multi-tenancy)
6. [Sessions & tokens](#6-sessions--tokens)
7. [Service-to-service auth](#7-service-to-service-auth)

---

## 1. Authentication methods

Keyed to what we protect (a USB-installed demo, fake credentials, no real PII) — not to the 18 Aug date.

| Method | Phase 1 | Later |
|--------|---------|-------|
| **Email + password** | **Now.** Two-step Welcome → email → password (doc 02). UX validates email format and non-empty password; **auth decision is the mock adapter**, not a screen `if`. | Same *shape* against the real API. |
| OAuth / social (Google, Apple, …) | **No** | Phase 3+ if the product needs it. Not an RN SDK in Phase 1. |
| SSO / SAML / OIDC | **No** | Backend / IdP concern when a real org exists. |
| Passwordless / magic link | **No** | Not in the migration template until a real mailer exists. |
| MFA | **No** | Phase 3+ when real accounts exist. |
| Biometrics (Face ID / fingerprint) | **No** (doc 15: internet-only permissions) | After real JWT; not a Phase 1 permission. |

**Credential source (Phase 1)**

- One shared demo pair: `DEMO_EMAIL` / `DEMO_PASSWORD` in gitignored `.env`.
- Repo commits `.env.example` placeholders only (doc 06).
- The **mock adapter** reads env and compares. Screens never hardcode the pair.

**Forecloses:** Auth0/Google/Apple Sign-In, SSO, magic links, MFA, and Face ID on 18 Aug.

---

## 2. Identity provider — build vs. buy

ADR-0009.

| When | Choice |
|------|--------|
| **Phase 1** | **No IdP.** Mock adapter inside the API module (ADR-0002). Demo credentials + mock JWT. |
| **Phase 3** | **Buy** a managed provider (Cognito, Auth0, Clerk, …). Exact vendor is **OQ-03** — backend team. |
| **Where it lives** | **Behind our API**, not as a React Native SDK. The app still `POST`s email/password to *our* contract and stores *our* JWT. |

Rolling our own password hashing, recovery, and MFA is the costly path this session exists to avoid. Putting Auth0/Firebase **in the app** would couple the client to a vendor and fight the API-module swap.

**Forecloses:** Auth0/Firebase/Cognito as a Phase-1 npm dependency; a homemade IdP in this repo; treating the mock adapter as production identity.

---

## 3. User / account model

Consistent with doc 04. **User** 1──1 **Session**. No orgs, no household profiles.

| Noun | Phase 1 |
|------|---------|
| **User** | `id` (JWT `sub`) + `userName` from `/me`. No email on the slice. Memory-only `getMe` cache. |
| **Session** | `accessToken` + `expiresAt`. Auth slice. The only persisted entity. |
| **Credentials** | Login request DTO `{ email, password }`. Never stored, never logged. |

**Lifecycle**

| Event | Phase 1 | Phase 3 |
|-------|---------|---------|
| Signup | **None.** One shared demo pair. | Managed IdP / backend. |
| Verification | **None.** | IdP. |
| Login | Mock adapter mints JWT. | Same client flow → our API. |
| Logout | Clear auth + user + content → Welcome. | Same. |
| Session expiry | Mock `exp` / `/me` 401 → Welcome. | Same client path; server may also revoke. |
| Recovery / forgot password | **None.** | IdP. |
| Deactivation / deletion | **None.** | IdP / backend. Uninstall is best-effort (Keychain may survive — doc 06). |

Household **profiles** stay deferred (doc 01). Do not stub signup/recovery screens for the demo.

**Forecloses:** self-serve registration, forgot-password, and multi-profile switching on 18 Aug.

---

## 4. Authorization model

ADR-0010.

**Binary:** session present or not. Unauthenticated navigator vs app navigator. No RBAC, no ABAC, no ownership checks, no per-title entitlements.

**Enforcement**

| Layer | What it does |
|-------|----------------|
| App shell | Auth selectors switch navigators. Cold-start loader until `getMe` + `getHomeFeed` + `getContinueWatching` complete. |
| Axios request interceptor | Attaches `Authorization: Bearer` on operations 2–5. Reads the token from the store (`injectStore`). |
| Axios response interceptor | Maps HTTP status → `ApiError.code`. Does **not** clear the session. |
| `baseQueryWithAuth` | **Reacts** to `UNAUTHORIZED` by clearing the session and `resetApiState()`; ignores `INVALID_CREDENTIALS`. (**ADR-0017** / **ADR-0020**). |
| Mock handlers (later: backend) | 401 without a valid token (`exp` + presence). |
| Screens / hooks | **Do not** check roles. Use generated RTK Query hooks only. |

No stub `role` / `entitlements` field on `/me`. Phase 3 can add claims **in the API contract** without rewriting the binary nav split. Parental controls and profiles wait until those features exist.

**Forecloses:** admin UI, kids vs adult, and “can play / can download” flags on 18 Aug.

---

## 5. Multi-tenancy

**None.** No tenant id on User, Session, or feed requests. Isolation is “this device’s JWT or not.”

A later B2B/org product is a new claim plus a backend filter — not a Phase 1 field we pretend to honor.

**Forecloses:** per-tenant fixtures, `X-Tenant-Id`, and org switchers on 18 Aug.

---

## 6. Sessions & tokens

Closes **OQ-25** (client mock). Production claims remain **OQ-03**.

| Topic | Phase 1 |
|-------|---------|
| Kind | **Access JWT only.** No cookies, no refresh token. |
| Lifetime | Mock `exp` = **7 days** from mint. Next successful login remints. |
| Storage | Auth slice → `redux-persist` + `react-native-encrypted-storage` (ADR-0003). Do not reopen. |
| Restore | Rehydrate → always `GET /me`. |
| Revocation | **Client-side only:** logout, `/me` 401, or `exp`. No denylist (no host). |
| Header | `Authorization: Bearer <jwt>` on authenticated calls (interceptor). Locked in doc 11 §9.1. |

**Mock JWT claims** (adapter; discarded with the mock in Phase 3):

| Claim | Value |
|-------|--------|
| `sub` | User `id` (UUID) |
| `exp` | Unix expiry (7 days) |
| `iat` | Unix issued-at, if the adapter signs |

Do **not** put `userName`, email, or roles in the token.

**`GET /me` body (closes OQ-25):** `{ id, userName }` only. **Path and JSON names locked in 1.11**
(doc 11 §5, §7.4). Note one wire detail decided there: the login response carries **`expiresAt` as
an ISO 8601 string**, not the JWT's numeric `exp` — so the client never decodes the token, and no
`jwt-decode` dependency is needed (doc 11 §6.4).

Phase 3 keeps this **client shape**. Refresh tokens and server revocation wait until a real IdP exists — do not add a mock refresh “for later.” DF3 still holds: when refresh arrives, it plugs into the API module / interceptor, not screens.

**Forecloses:** cookie sessions, refresh rotation, and stuffing profile into the JWT on 18 Aug.

---

## 7. Service-to-service auth

**None.** One RN process, in-process mocks. Auth → API module is an in-process import via middleware (doc 03). No mTLS, service JWTs, or client-held API keys between components.

When Phase 3 adds a host, **the app authenticates as a user** (JWT on HTTPS), not as a service. A later BFF/service mesh is a new design.

**Forecloses:** a client-held service credential and a mock HTTP server just to have mTLS.

---

## Decision Summary

| # | Decision | Choice | Rationale | Forecloses / tradeoff |
|---|----------|--------|-----------|-----------------------|
| 1 | Auth methods | Email + password now; OAuth/SSO/passwordless/MFA/biometrics later | Asset is a fake demo login, not real accounts | Social/SSO/Face ID on 18 Aug |
| 2 | IdP | Phase 1: mock adapter. Phase 3: **buy**, behind our API, vendor TBD (OQ-03) | Avoid homemade identity and RN-SDK lock-in | Auth0/Firebase in the app; rolling our own IdP |
| 3 | Account model | 1:1 User↔Session; login + logout + expiry only | Matches doc 04; one shared demo pair | Signup, recovery, profiles on 18 Aug |
| 4 | Authorization | Binary session gate; no roles on `/me` | Nothing to privilege-escalate into | Admin, entitlements, kids mode |
| 5 | Multi-tenancy | None | One client, one demo account | Tenant headers / org switcher |
| 6 | Sessions | Access JWT, 7-day `exp`, no refresh; claims `sub`/`exp`/`iat`; `/me` = `{ id, userName }` | Closes OQ-25; DF3 refresh still a Phase 3 plug-in | Cookie sessions; mock refresh; profile in JWT |
| 7 | S2S | None | One process; user JWT when a host exists | Client service credentials |

## Open Questions

| ID | Question | Owner | Feeds into |
|----|----------|-------|------------|
| OQ-03 | Production backend contract and JWT claims shape (vendor + real claims) | Backend team | Doc 11 §14 item 3; Phase 3 |
| ~~OQ-22~~ | ~~JSON names and paths for `/me`, login, and the JWT header~~ **Resolved (1.11):** doc 11 §5, §7.4, §9.1 | — | closed |

~~OQ-25~~ **Resolved (1.6a):** mock JWT claims = `sub` + `exp` + `iat`; `/me` body = `{ id, userName }`.

**Added by 1.11 (doc 11 §8.4, §9.2), because both bear directly on this doc's flows:**

- The session-clearing **401 reaction excludes login** — otherwise a wrong
  password would bounce the user out of the credentials screen and destroy the F2 inline-error
  demo. `INVALID_CREDENTIALS` and `UNAUTHORIZED` are distinct codes so the distinction is
  mechanical. The reaction lives in **`baseQueryWithAuth`** (**ADR-0017** / **ADR-0020**) — *not* in the axios
  interceptor. The request interceptor attaches the header; the response interceptor maps status → `code`.
- The **mock handlers validate the token's presence *and* `exp`** on authenticated operations
  (and they see the interceptor-attached header because mocks use `axios-mock-adapter` on the same instance).

Carried: privacy session remains Deferred (doc 04). Pinning / root detection remain **RISK-0007**. Shared demo login remains **RISK-0008**.

## Version Log

| Version | Date | STEP | Change |
|---------|------|------|--------|
| v0.1.0 | 2026-08-17 | STEP-1.6a | Initial identity & auth design. ADR-0009, ADR-0010. Closed OQ-25. |
| v0.1.1 | 2026-08-17 | STEP-1.11 | §6 header and `/me` names locked in doc 11; `expiresAt` is ISO on the wire. Added the 401-scoping rule and the mock's token-validation obligation. Closed OQ-22. |
| v0.1.2 | 2026-08-17 | STEP-1.12 | **ADR-0017:** the session-clearing 401 *reaction* moves out of the axios interceptor to the transport-agnostic middleware / API-module error handling, so Phase 1 actually executes and tests it. §4's enforcement table gains that row and narrows the interceptor's. No change to auth methods, claims, lifetimes, storage, or the authorization model. |
| v0.2.0 | 2026-08-17 | STEP-1.14 | Token lifecycle is RTK Query `baseApi` + axios interceptors (`injectStore`); 401 reaction in `baseQueryWithAuth`; `/me` is cache not a user slice (**ADR-0020**). |
