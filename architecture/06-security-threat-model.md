# Doc 06 — Security & Threat Model

**Version:** v0.1.2
**Status:** Draft
**Last updated:** 2026-08-17 (STEP-1.8)
**Audience:** Mobile developers, QA, backend team (Phase 3)

> What this React Native demo must protect, where trust is crossed, which threats we mitigate in Phase 1, and which we accept until a real host, real accounts, or Bitrise exists.

This is an **abbreviated** threat model matched to an internal USB-installed POC with in-process mocks and fake fixtures — not a skipped one. Session 1.6 is **Done**. Network-grade controls wait for Phase 3; dependency CI waits for Bitrise (1.8). Identity deep-design is **Done** in `architecture/16-identity-auth.md`. Privacy/compliance stays **Deferred** until Phase 3 real accounts (doc 04).

## Table of Contents

1. [Assets](#1-assets)
2. [Trust boundaries](#2-trust-boundaries)
3. [Threats → mitigations](#3-threats--mitigations)
4. [AuthN / AuthZ posture](#4-authn--authz-posture)
5. [Secrets & data protection](#5-secrets--data-protection)
6. [Web / client-risk posture](#6-web--client-risk-posture)

---

## 1. Assets

Phase 1 has **no real people, payments, or public API**. Treat the *shapes* as production so they are not logged or leaked.

| Asset | Why it matters | Phase 1 reality |
|-------|----------------|-----------------|
| Login credentials (email + password) | Impersonation of the demo session | Request DTO only — never stored |
| Session JWT (`accessToken` + `exp`) | Unlocks `/me`, HomeFeed, Continue Watching | Encrypted persist; 7-day mock TTL |
| `userName` (and JWT `sub`) | PII-shaped identity | Memory-only; fake fixture |
| Continue-watching progress | Per-session viewing data | Fixture; confidential in the real product |
| Catalog fixtures / API contract | Demo integrity | In-process mocks |
| Demo availability (18 Aug sign-off) | The actual blast radius today | Local installs, handful of devices |
| Source / architecture as a migration template | Team IP | Repo hygiene; no secrets in git |

**Out of the asset list:** real PII, payments, health data, children’s accounts, production content licenses, a public API, a backend to DDoS.

**Privacy/compliance:** still **Deferred**. Revisit **before Phase 3 real accounts / real JWT**. This session covers keeping attackers *out*; lawful processing is not in scope until there are data subjects.

---

## 2. Trust boundaries

```
 Human / stakeholder
        │ B1 (UI)
        ▼
 ┌──────────────────────────────────────┐
 │           RN app process             │
 │  Shell · Auth · Storefront · Shared  │
 │                 │ B3                 │
 │                 ▼                    │
 │          API module (mocks)          │
 └───────┬──────────────────┬───────────┘
         │ B2               │ B4 (Phase 3)
         ▼                  ▼
 OS secure storage     future HTTPS backend
         │ B5
         ▼
 device OS / other apps / screenshots

 B6: workstation ↔ git (dev-time)
```

| # | Boundary | What crosses it | Live in Phase 1? |
|---|----------|-----------------|------------------|
| **B1** | Human ↔ app UI | Email/password, `userName`, catalog, tap-alerts | Yes |
| **B2** | App process ↔ OS secure storage | JWT + `exp` via `react-native-encrypted-storage` | Yes |
| **B3** | Features ↔ API module | Login DTO, JWT, `/me`, HomeFeed, Continue Watching | Yes (in-process mocks; same shapes as later HTTP) |
| **B4** | API module ↔ future backend | Same payloads over TLS | **No** — Phase 3 swap |
| **B5** | App ↔ device OS / other apps | Screenshots, backups, clipboard, debug/jailbreak | Yes (weak: local demo devices) |
| **B6** | Repo / workstation ↔ source | Fixtures, demo credentials, `.env` | Yes (dev-time) |

**Not boundaries in Phase 1:** user↔admin (no admin), service↔service (one process), app↔third party (no IdP, analytics vendor, payments, CDN).

B3 and B4 are the same *logical* contract; B4 is the one that becomes a real network attack surface.

---

## 3. Threats → mitigations

STRIDE-lite. Privilege escalation is **out** until there are roles. Insider misuse is **B6 only** (the shared demo account is the product, not a bug).

| Threat | Boundary | Mitigation | Deferred / blast radius |
|--------|----------|------------|-------------------------|
| Spoofing — anyone with the shared demo login is “the user” | B1 | **Accept.** One demo account is the design. Auth still goes through the mock adapter, not a screen `if`. | Radius: internal demo. Trigger: Phase 3 real accounts (swap designed in doc 16 / ADR-0009). **RISK-0008** |
| Tampering — garbage email/password | B1 | UX validation (email format, non-empty password). **Auth decision is the mock adapter**; same error shape as the future API. Password field uses secure text. | — |
| Disclosure — on-screen `userName`, titles | B1 | Intended. Do not log them from analytics stubs. | — |
| Disclosure — leftover JWT after uninstall | B2 | **Accept** OS Keychain survival. `/me` 401 / `exp` still clears the session when the app runs. | Radius: leftover mock JWT on reinstall. Trigger: real JWT / store release. |
| Tampering — corrupt persist blob | B2 | redux-persist `version: 1`; incompatible shape → **purge** (doc 04). | — |
| Replay — restore an old JWT | B2 | Mock `exp` + `/me` on every cold start; 401 / expiry → Welcome. No refresh token. | — |
| Disclosure — JWT / password in logs | B3 | Never log password, JWT, or full `/me`. Analytics stubs: no PII in payloads (doc 04). | — |
| Tampering — screens/hooks calling axios | B3 | Import rule (doc 03): API I/O only via middleware. | — |
| Stolen JWT on the wire, MITM, injection, replay vs a server, backend DoS | B4 | **Defer.** No host exists. | Radius: none until a real host. Trigger: **Phase 3**. **RISK-0009**. Pinning already **RISK-0007**. |
| Disclosure — screenshots, debug builds, USB device theft | B5 | **Accept.** Secure-text on the password field is the now-control. No pinning / root detection (doc 15). | Radius: whoever holds the demo phone. Trigger: store-bound or real PII. |
| Disclosure — demo password in git | B6 | Demo pair lives in gitignored `.env`. Repo commits `.env.example` placeholders only. Mock adapter reads env. | — |
| Tampering — dependency supply chain | B6 | Commit the lockfile. Vet before add (`runbooks/dependency-supply-chain.md`). `npm audit` on the check-in cadence. | **`npm audit` in CI still deferred** — 1.8 left the gate undecided until Bitrise exists. **RISK-0010**. |
| Rate limiting / brute force of the demo login | B4 (future) | **Defer.** No public endpoint; brute force ≡ reading `.env.example`. | Radius: none. Trigger: **Phase 3 backend**. **RISK-0009**. |

XSS, CSRF, clickjacking, CORS: **N/A** (native client, JWT on an axios header when B4 exists, no cookies, no WebView).

---

## 4. AuthN / AuthZ posture

High-level stance. Deep design is **`architecture/16-identity-auth.md`** (1.6a **Done**).

**AuthN — who you are**

- Email + password against the **mock adapter** (`DEMO_EMAIL` / `DEMO_PASSWORD` from `.env`).
- Success mints a **mock JWT** (`sub` + `exp` + `iat`; 7-day `exp`, remint on next login). **No refresh token.**
- Session restore from encrypted persist; cold start always **`/me`**. 401 / expiry → clear auth + user + content → Welcome.
- One shared demo account. No signup, verification, recovery, MFA, or SSO in Phase 1.
- Phase 3: **buy** a managed IdP **behind our API** (ADR-0009). Vendor TBD (OQ-03).

**AuthZ — what you may do**

- **Binary:** session present or not. Unauthenticated navigator vs app navigator (ADR-0010).
- JWT on `/me`, HomeFeed, and Continue Watching. `/me` body is `{ id, userName }`. No roles, no admin, no household profiles, no per-title entitlements.

**Forecloses:** Auth0/Firebase on 18 Aug; RBAC now; treating “hardcoded password in the screen” as the auth design.

---

## 5. Secrets & data protection

**At rest**

- JWT: `redux-persist` + `react-native-encrypted-storage` (Keychain / EncryptedSharedPreferences). ADR-0003. Do not reopen.
- Email/password: request lifetime only. Never persist, never log.
- `userName` / catalog: memory only.

**In transit**

- Phase 1: N/A (in-process mocks).
- Phase 3 (B4 live): **HTTPS only**, iOS ATS on, no cleartext exception. Certificate pinning stays deferred (**RISK-0007**).

**Where secrets live (never in git)**

- Local: gitignored `.env` (values) / `.secrets/` (files). Repo commits only `.env.example` with placeholders (`templates/env-example.txt`).
- Phase 1 keys in `.env.example`: `API_BASE_URL`, `DEMO_EMAIL`, `DEMO_PASSWORD`. The mock adapter reads the demo pair from env — **not** a committed fixture and **not** a screen-level `if`.
- Production / CI secrets manager: **none until** Bitrise (1.8) or a real backend (Phase 3). No Vault/AWS SM for 18 Aug.

**Rotation**

- Mock JWT: 7-day `exp`, remint on next login. No refresh.
- Demo password: no cadence. If it appears in a public gist/chat, change `.env` and tell stakeholders. Suspected leak of anything *real* → `runbooks/secrets-rotation.md` Part 2.
- Mock JWT signing key (if the adapter even signs): dummy in-process constant; discarded with the mock in Phase 3.

---

## 6. Web / client-risk posture

| Risk | Phase 1 posture | Who owns it |
|------|-----------------|-------------|
| **Input validation** | Email format + non-empty password on the credentials screen (UX). Auth decision is the mock adapter. UI validation is not a security control. | Auth feature + API module |
| **Injection** | No SQL, no HTML, no `eval`, no WebView. Don’t build URLs/file paths from user input. Artwork URIs are bundled placeholders. | API module / Storefront |
| **XSS / CSRF / clickjacking / CORS** | **N/A.** | — |
| **Rate limiting / brute force** | **Defer** to Phase 3 backend. | **RISK-0009** |
| **Logging / PII leak** | Never log password, JWT, or full `/me`. Analytics stubs: no PII. | API interceptor + analytics stub |
| **Dependencies** | Lockfile committed. Vet before `npm install`. `npm audit` on the check-in cadence; deferrals go in `risks.yml`. Security patches promptly; majors deliberately. Hard deps already locked (doc 03). Do not add Firebase/Sentry “for security.” | Every STEP that adds a package |
| **`npm audit` in CI** | **Still deferred.** Session 1.8 reviewed it and **consciously did not decide** the gate: a fail-vs-warn threshold is meaningless with no CI to enforce it. Revisit at **Bitrise implementation** (doc 08 §5). | **RISK-0010** (OQ-27 closed) |

Operational runbooks: `runbooks/secrets-rotation.md`, `runbooks/dependency-supply-chain.md`.

---

## Decision Summary

| # | Decision | Choice | Rationale | Forecloses / tradeoff |
|---|----------|--------|-----------|-----------------------|
| 1 | Assets | Credentials, JWT, PII-shaped `userName`, CW progress, fixtures/contract, demo availability, source as template | Matches docs 03/04/15; no real PII or money | Treating Disney-lookalike art as a legal asset here (that’s RISK-0005) |
| 2 | Trust boundaries | B1–B6; B4 named but not live; no admin/third-party/s2s | Threats live on lines of trust; B3/B4 share the contract | Inventing a mock HTTP server just to have a network edge |
| 3 | STRIDE map | Spoofing/disclosure/tampering/replay as in §3; no priv-esc; insider = B6 only | Realistic for an internal mock client | Generic web STRIDE dump |
| 4 | AuthN/AuthZ | Binary mock JWT gate; 1.6a Done (doc 16); privacy stays Deferred | Login is Phase 1a; no roles yet | IdP or RBAC on 18 Aug |
| 5 | Secrets | `.env` / `.env.example`; encrypted JWT at rest; TLS when B4 exists; no secrets manager yet | Train the migration template; blast radius is still a USB demo | Committing the demo password in fixtures |
| 6 | Client risks | Validate UX-only; no XSS/CSRF; lockfile + vet now; CI audit at Bitrise | RN is not a browser; no public endpoint | WAF/rate-limit for 18 Aug |
| 7 | Mitigate vs defer | Now = logging/storage/import rules/`.env`/lockfile. Defer = B4 STRIDE, rate limit, CI audit, pinning. Accept = shared login, Keychain uninstall, device theft | Fortress ≠ this blast radius; deferral is recorded | Skipping 1.6 entirely |
| 8 | Session disposition | **1.6 Done** (abbreviated complete), not Deferred | Conscious model, not an oversight. ADR-0008 | Reopening network controls before a host exists |

## Open Questions

| ID | Question | Owner | Feeds into |
|----|----------|-------|------------|
| ~~OQ-27~~ | ~~Exact dependency-audit gates once Bitrise exists (`npm audit` fail vs warn, lockfile check, cadence)~~ **Closed (1.8) as deliberately deferred:** undecidable without a CI system; RISK-0010's revisit trigger moves to Bitrise implementation | — | closed |
| ~~OQ-25~~ | ~~Exact mock `/me` payload beyond `id` + `userName` (claims vs body)~~ **Resolved (1.6a):** mock JWT = `sub` + `exp` + `iat`; `/me` = `{ id, userName }`. JSON names → OQ-22 | — | closed |
| OQ-03 | Production backend contract and JWT claims shape | Backend team | 1.11 Interface Contracts; Phase 3 |

Carried: privacy session remains Deferred (doc 04). Pinning / root detection remain **RISK-0007**. Identity living doc is `architecture/16-identity-auth.md`.

## Version Log

| Version | Date | STEP | Change |
|---------|------|------|--------|
| v0.1.0 | 2026-08-17 | STEP-1.6 | Initial abbreviated threat model. ADR-0008. RISK-0008–0010. |
| v0.1.1 | 2026-08-17 | STEP-1.6a | Identity deep-design Done (doc 16). Closed OQ-25. OQ-03 now 1.11 / Phase 3. |
| v0.1.2 | 2026-08-17 | STEP-1.8 | §6 CI audit gate reviewed and consciously left undecided; closed OQ-27, RISK-0010 revisit moves to Bitrise implementation. Secrets posture unchanged; doc 08 §5 adds the `.env`-on-build-machine pre-flight check. |
