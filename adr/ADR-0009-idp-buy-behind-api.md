# ADR-0009: Phase-1 mock auth; Phase-3 managed IdP behind our API

**Status:** Accepted
**Date:** 2026-08-17

## Related documents
- `architecture/16-identity-auth.md`
- `architecture/03-architecture-overview.md` (§7 build vs. buy)
- `architecture/06-security-threat-model.md` (§4 AuthN/AuthZ)
- ADR-0002 (in-app API module; no BaaS in Phase 1)
- OQ-03 (production vendor and JWT claims)

## Context

Login is in Phase 1a, so identity cannot stay undecided. The default for real accounts is a
managed identity provider — rolling your own hashing, recovery, and MFA is a common and
costly mistake. This project has no real accounts, no backend, and a locked API-module
swap (ADR-0002). Putting Auth0 or Firebase in the React Native app would couple the client
to a vendor before a host exists. Treating the mock adapter as a homemade IdP would train
the wrong production path.

## Decision

1. **Phase 1: no identity provider.** Auth is the **mock adapter** inside the API module.
   One shared demo pair from gitignored `.env`. The adapter mints a mock JWT. Screens do
   not compare passwords.
2. **Phase 3: buy a managed IdP** (Cognito, Auth0, Clerk, or equivalent). Do **not** roll
   our own identity in this repo or in a backend we do not own.
3. **The buy sits behind our API**, not as a React Native SDK. The app continues to post
   email/password to *our* contract and store *our* JWT. Vendor and production claims stay
   **OQ-03** for the backend team.

**Rationale.** Phase 1 has nothing worth federating. The migration template must look like
a client of *our* API, because that is what Phase 3 swaps. Buying later is cheap if the
client never imported an IdP SDK. Homemade identity owns every recovery and incident path.

**Alternatives.** Auth0/Firebase SDK in the app now — rejected; no BaaS in Phase 1
(ADR-0002) and it breaks the mock-first contract. Homemade IdP “so we own it” — rejected;
this team is not standing up identity. Defer the whole build-vs-buy call — rejected; login
ships on 18 Aug and 1.6a exists to lock the swap path.

**Reversibility.** High for the Phase 3 vendor (behind the API). Low for an RN IdP SDK or
password checks in screens.

## Consequences

- `architecture/16-identity-auth.md` is the living identity design.
- Phase 3 integration is a mock-adapter drop plus `baseURL` — not an Auth0 lock-in rewrite.
- Backend team picks the vendor (OQ-03); mobile does not.
- RISK-0008 (shared demo login) stays open until Phase 3 real accounts.
