# ADR-0010: Binary session gate; no roles or entitlements in Phase 1

**Status:** Accepted
**Date:** 2026-08-17

## Related documents
- `architecture/16-identity-auth.md`
- `architecture/06-security-threat-model.md` (§4 AuthN/AuthZ)
- `architecture/03-architecture-overview.md` (shell switches navigators)
- `architecture/04-data-model.md` (User + Session)

## Context

Authorization (“what may you do?”) is easy to fake with a stub `role` on `/me` and then
have screens branch on it. Phase 1 has one shared demo user, no admin, no household
profiles, and no per-title catalog rights. Privilege escalation is not a threat until
there is a privilege. A stub field would look like a contract the backend never agreed to.

## Decision

1. **Authorization is binary:** a session is present or it is not.
2. **Enforcement** is the app shell (auth selectors → unauthenticated vs app navigator),
   the API interceptor (JWT on `/me`, HomeFeed, Continue Watching), and the mock adapter
   (401 without a valid token). Screens and hooks do not check roles.
3. **No `role` / `entitlements` field on `/me`.** Phase 3 may add claims in the API
   contract without rewriting the navigator split.
4. **No multi-tenancy.** No tenant id on User, Session, or feed requests.

**Rationale.** There is nothing to authorize beyond “logged in.” A stub role trains
screens to do AuthZ, which is the opposite of the interceptor/adapter split. Profiles,
parental controls, and entitlements are later features (doc 01) — design them when they
exist.

**Alternatives.** RBAC with a dummy `user` role — rejected; unused contract surface.
ABAC / entitlement flags on cards — rejected; no catalog rights in Phase 1. Put
`userName` or roles in the JWT — rejected; profile stays on `/me` (OQ-25 closed in
doc 16).

**Reversibility.** High for adding claims later (contract change). Low if screens start
switching on a fake role.

## Consequences

- `/me` stays `{ id, userName }`.
- Admin UI, kids vs adult, and “can download” flags are out of Phase 1.
- Session 1.11 does not invent a permissions envelope.
- Doc 06’s “no priv-esc until there are roles” still holds.
