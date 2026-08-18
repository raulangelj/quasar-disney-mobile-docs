# ADR-0017: The session-clearing 401 policy lives above the transport, not in the interceptor

**Status:** Accepted
**Date:** 2026-08-17

## Related documents

- `architecture/11-interface-contracts.md` §8.3 (`ApiError` normalization), §8.4 (the 401 collision), §6.5 (transport profiles)
- `architecture/12-test-strategy.md` §3.1 (the must-cover 401-scoping test), §2 (why T2 exists)
- `architecture/16-identity-auth.md` §4, §6 (enforcement layers; session revocation)
- `architecture/02-phasing-roadmap.md` §4 (criterion **F2**)
- ADR-0002 — In-app API module with axios; no backend component
- ADR-0016 — TypeScript interfaces are the contract now

## Context

Doc 11 §8.4 names a trap worth naming: a login failure and an expired token are both `401`. The
reaction to the second — clear the session and return to Welcome — must not fire on the first,
because if it does, a wrong password bounces the user out of the credentials screen and **criterion
F2's inline error state never renders**. F2 is roughly half of what the 2026-08-18 demo exists to
show (doc 02 §4).

Doc 11 §8.4 states the rule and assigns it a home:

> *"The **interceptor's** session-clearing 401 handler is scoped to authenticated operations and
> explicitly excludes `/auth/login`."*

Session 1.12 found that the placement and the Phase-1 reality disagree. Doc 11 §6.5 and OQ-17
establish that Phase 1's transport is the **mock adapter**, sitting behind the same functions as the
axios client; the axios instance is *not* mocked, and `API_BASE_URL` is inert (doc 09 §4.2). No
Phase-1 request passes through an axios interceptor at all.

Read literally, then, the rule protecting F2 lives in a code path **Phase 1 never executes**. It
would run for the first time in Phase 3, against a real backend — which is precisely the situation
doc 11 §9.2 already rejected for the token-validation obligation, on the grounds that leaving a
branch untested until first contact with a real server is how it fails there.

Three placements were available:

**A. Keep it in the interceptor.** Honest to how a production HTTP client is usually written, and
wrong for Phase 1: untestable, unexercised, and it makes doc 12's must-cover 401-scoping test
impossible to write.

**B. Duplicate it** — once in the adapter, once in the interceptor. Two implementations of one
security-relevant rule, guaranteed to drift, and doc 11 §1.1 exists to prevent exactly this pattern
for exactly this reason.

**C. Move the policy above the transport**, into the layer that consumes the already-normalized
`ApiError`.

## Decision

**1. The 401 policy is transport-agnostic and lives above the transport** — in the middleware / API
module error handling that consumes the normalized `ApiError` (doc 11 §8.3).

**2. It switches on `code`, never on a path comparison.** `UNAUTHORIZED` clears the session;
`INVALID_CREDENTIALS` does not. Doc 11 §8.2 already made these distinct codes so that this
distinction would be mechanical rather than a string match against `/auth/login`.

**3. The axios interceptor's remaining job is narrower and unchanged in substance:** attach
`Authorization: Bearer <jwt>` on authenticated operations (doc 16 §6), and map HTTP status → `code`
when a real transport exists. It does not own the reaction.

**Rationale.** Doc 11 §8.3 already normalizes both transports into one `ApiError` *above* the
transport, so a policy that keys on `ApiError.code` is the only one both transports can share
without duplication. That makes the rule exercised by the mock path today, which in turn makes doc 12
§3.1's T2 test real rather than aspirational — the test that protects F2 now runs on every push
(doc 12 §7.1) instead of waiting for Phase 3.

**Alternatives.** *A* was rejected because it leaves the F2-protecting branch dead in Phase 1. *B*
was rejected on doc 11 §1.1's grounds: one contract, stated once — a security-relevant reaction
implemented twice will diverge, and the divergence will be discovered by whichever half is not the
one under test.

**Reversibility.** Low cost. The policy is a `switch` on an enum in one place; relocating it later is
mechanical. Nothing in the wire contract changes — this ADR moves no field, renames nothing, and
alters no payload.

## Consequences

**Easier.** The rule is testable in Phase 1a with the mock adapter as the transport, so doc 12's
401-scoping test — one of the highest-value tests in the suite, since it guards half of F2 — becomes
a T2 integration test in the merge gate. The Phase-3 swap also gets *simpler*: an interceptor that
only attaches a header and maps status codes has less to get wrong than one carrying session policy.

**Harder.** It departs from the common idiom, where an axios interceptor handles 401 globally. A
developer arriving from another React Native codebase will look for the logic in the interceptor and
not find it. Doc 11 §8.4 and doc 12 §3.1 both point at the real location, and the interceptor should
carry a comment saying where the policy lives and why (per the project's docstring rule).

**No new risk.** The rule's *content* is unchanged from doc 11 §8.4 — only its home. What changes is
that Phase 1 now executes and tests it.

**Doc 11 is amended, not superseded.** §8.4's paragraph is corrected and its Version Log bumped;
Decision Summary row 19 ("401 scoping") stands as written, since the decision was always *that* the
session-clearing reaction excludes login. Doc 16's closing note, which restated §8.4's wording, is
corrected to match.

## Amendment (2026-08-17 — STEP-1.14 / ADR-0020)

The policy is unchanged: switch on `code`; `UNAUTHORIZED` clears the session; `INVALID_CREDENTIALS`
does not; the raw axios interceptor does not own the reaction.

**The home is now `baseQueryWithAuth`**, the RTK Query wrapper around `axiosBaseQuery`, not
handwritten feature middleware. On `UNAUTHORIZED` it also `resetApiState()`.

**Mocks now go through the same axios instance** (`axios-mock-adapter`, ADR-0020). The original
reason this ADR moved the policy — Phase 1 never executed interceptors — no longer applies to
*header attach*. Session-clear still stays out of the raw interceptor so F2 cannot be broken by a
global 401 handler. The request interceptor attaches `Authorization`; the response interceptor maps
status → `ApiError`; `baseQueryWithAuth` reacts.
