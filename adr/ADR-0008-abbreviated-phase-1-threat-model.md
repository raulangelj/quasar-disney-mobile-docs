# ADR-0008: Abbreviated Phase-1 threat model with recorded deferrals

**Status:** Accepted
**Date:** 2026-08-17

## Related documents
- `architecture/06-security-threat-model.md`
- `architecture/04-data-model.md` (§5 PII; privacy session Deferred)
- `architecture/15-native-app-architecture.md` (§6 device security)
- ADR-0003 (auth slice in secure storage)
- RISK-0007, RISK-0008, RISK-0009, RISK-0010

## Context

Session 1.6 can end as a *deliberate* deferral of the whole threat model, or as a complete
model whose controls match blast radius. This project is an internal React Native POC:
in-process mocks, fake fixtures, local Xcode/Android Studio installs, no public endpoint,
no real PII, sign-off on 2026-08-18.

Doc 02 already called for 1.6 to run abbreviated. The risk of *skipping* 1.6 is that
logging, `.env` hygiene, and import rules never get written down and the migration template
ships the wrong habits. The risk of a fortress is burning the remaining architecture
sessions on MITM and rate limits that have no host to attach to.

## Decision

1. **Complete session 1.6 as Done** — abbreviated, not skipped. Mark the substep `Done`,
   not `Deferred`.
2. **Mitigate now:** never log password/JWT/`/me`; encrypted persist of the auth slice
   (ADR-0003); persist purge + `/me` 401 / `exp` → login; auth via the mock adapter, not a
   screen `if`; screens/hooks never call axios; demo credentials in gitignored `.env` with
   committed `.env.example` only; lockfile + vet-before-add; secure-text password field.
3. **Defer until Phase 3 (real host / real accounts):** network STRIDE on B4 (MITM, stolen
   JWT on the wire, injection, replay vs a server, backend DoS), rate limiting / brute
   force, certificate pinning and jailbreak/root detection (already RISK-0007),
   privacy/compliance (already doc 04).
4. **Defer until Bitrise (session 1.8 / Phase 2):** `npm audit` (and related lockfile
   gates) **in CI**. Local vet + check-in-cadence audit still apply. Exact fail-vs-warn
   policy is OQ-27.
5. **Accept:** shared demo login as the product (RISK-0008); iOS Keychain surviving
   uninstall; screenshots / debug builds / USB device theft.
6. **1.6a Identity & Auth still runs** (login is Phase 1a). Binary AuthN/AuthZ is the
   Phase-1 stance; 1.6a owns claims, `/me`, and the Phase 3 IdP call.

**Rationale.** Anything on a public internet gets probed; this app is not on one yet. The
controls we keep are the ones that survive a backend swap and teach the template. Every
deferral has a blast radius and a trigger in `registries/risks.yml`.

**Alternatives.** Defer the entire 1.6 session — rejected; logging and secrets hygiene
would be unwritten. Full network threat model and pinning now — rejected; no host, and
doc 15 already deferred device MITM controls. Commit the demo password in fixtures —
rejected; it trains the wrong habit even though the blast radius is small.

**Reversibility.** High for deferred controls (add interceptors, CI jobs, an IdP later).
Low for “password in the screen” or AsyncStorage — those are the rewrites this session
exists to prevent.

## Consequences

- `architecture/06-security-threat-model.md` is the living posture.
- RISK-0008 (shared demo login), RISK-0009 (no server-side controls until Phase 3),
  RISK-0010 (CI audit waits for Bitrise) index the deferrals.
- Session 1.8 must reopen OQ-27 when Bitrise is designed.
- Session 1.6a must not re-litigate encrypted token storage or the binary gate; it
  designs the swap *off* that gate.
