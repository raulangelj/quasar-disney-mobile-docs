# ADR-0002: In-app API module with axios; no backend component

**Status:** Accepted
**Date:** 2026-08-16

## Related documents
- `architecture/03-architecture-overview.md`
- `architecture/02-phasing-roadmap.md` (DF1, DF2)
- `architecture/01-system-overview.md`

## Context

Phase 1 has no real backend. Features still need loading and error paths that are real, not
decorative, and a later backend integration must be a localized swap. Listing a “Backend API”
as a top-level component implied we own a server we will not build.

## Decision

1. **The API module is in-app code**, not a service. It owns wire types, the HTTP client,
   and the mock adapter.
2. **Transport is axios**, as a single configured instance (base URL, auth-header
   interceptor, error mapping) **inside the API module only**. Screens, hooks, and features
   never import `axios` or `fetch`.
3. **Features reach the API module only through Redux middleware.**
4. **Mocks return Promises with artificial latency and injectable failure**, carrying the
   exact shapes expected of a future backend.
5. **A future HTTP backend is not a component of this system.** Phase 3 swaps `baseURL` and
   drops the mock; callers stay.

**Rationale.** The product under design is the mobile app. A mock HTTP process would add
ops without making the swap clearer. Axios centralizes interceptors so a real token later
does not scatter auth headers through screens.

**Alternatives.** `fetch` from the API module — rejected; the team wants axios’s instance
and interceptor model. Calling axios from screens/hooks — rejected; that destroys the swap
boundary. A running mock server (MSW-as-process, json-server) — rejected for Phase 1.
Firebase/Auth0 as the “API” — rejected; no BaaS in Phase 1.

**Reversibility.** Low for the boundary (keep it). Medium for axios vs another client
(confined to the API module).

## Consequences

- Session 1.11 authors the contract against this module’s types, not against a service we
  host.
- Test Strategy can unit-test the mock adapter and middleware without a network.
- Backend-team integration is gated on their delivery, not on us scaffolding a server.
