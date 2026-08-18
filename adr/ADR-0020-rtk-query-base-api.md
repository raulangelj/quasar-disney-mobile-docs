# ADR-0020: RTK Query baseApi; axios interceptors attach headers

**Status:** Accepted
**Date:** 2026-08-17

## Related documents
- `architecture/03-architecture-overview.md` §4–§8
- `architecture/11-interface-contracts.md` §6.5, §8.4, §9
- `architecture/16-identity-auth.md`
- ADR-0002 (axios in the API module — amended by this ADR)
- ADR-0017 (401 policy above the raw interceptor — still true; home is `baseQueryWithAuth`)
- ADR-0003 (persist the auth slice)

## Context

Docs 01/03 locked **Redux Toolkit slices plus explicit async middleware** so screens never call
axios, and so the API boundary stays visible (not hidden behind `createAsyncThunk`). Token
lifecycle lived in that middleware: login writes the JWT, the (future) interceptor attaches it,
`UNAUTHORIZED` clears the session.

The intended stack is **RTK Query** for APIs and token lifecycle: one `baseApi`, feature
`injectEndpoints`, and interceptors on the shared axios instance that attach `Authorization`.
Explicit custom middleware for every operation duplicates what RTK Query already is.

Phase 1 still has no real HTTP. If mocks sit *beside* axios (OQ-17), interceptors never run
until Phase 3 — the failure mode ADR-0017 was written to prevent. Putting mocks on the **same**
axios instance makes the interceptor path real in 1a.

## Decision

1. **All I/O goes through one RTK Query `baseApi`** (`createApi` in `src/api/baseApi.ts`) with
   empty endpoints. Features **`injectEndpoints`**: auth owns `login` and `getMe`; storefront owns
   `getHomeFeed`, `getContinueWatching`, and `getContainerResources`. Shell registers
   `baseApi.reducer` and `baseApi.middleware`.

2. **The HTTP engine is still axios** (ADR-0002), as a single instance in `src/api/client/`.
   **Request interceptor:** attach `Authorization: Bearer <jwt>` on operations 2–5, reading the
   token from the store (the shell calls `injectStore(store)` after `configureStore`).
   **Response interceptor:** map HTTP status → `ApiError { code, status, message }`. The
   interceptor does **not** clear the session.

3. **`baseQueryWithAuth` wraps `axiosBaseQuery`.** On `UNAUTHORIZED` it dispatches auth-slice
   clear **and** `baseApi.util.resetApiState()`, then returns the error. On
   `INVALID_CREDENTIALS` it does neither. It switches on `code`, never on a path. This is the
   same policy as **ADR-0017**, living in the RTK Query wrapper rather than handwritten
   middleware.

4. **Phase 1 mocks use `axios-mock-adapter` on that same instance**, with artificial 400–600 ms
   latency and injectable failure. Interceptors run. Token presence/`exp` is still enforced
   (doc 11 §9.2), now by the mock handlers plus the interceptor-supplied header. This **reverses
   OQ-17** (separate mock adapter that bypassed axios). Phase 3: drop the adapter, set
   `API_BASE_URL`; callers (RTK Query hooks) stay.

5. **Token lifecycle.** The **auth slice** still holds `accessToken` + `expiresAt` and is the
   only persisted slice (ADR-0003). `login.fulfilled` writes the slice; logout / `UNAUTHORIZED`
   clears it and resets the API cache. Catalog and `/me` live in the **RTK Query cache**
   (memory-only). There is no content slice and no user slice. Cold-start still waits on
   `getMe` + `getHomeFeed` + `getContinueWatching` before first paint.

6. **Import rules.** Screens and feature UI never import `axios`, `fetch`, or `src/api/client/`.
   Features import **their own** generated hooks (`useLoginMutation`, `useGetHomeFeedQuery`, …)
   and `src/api/types/` only. Features do not import another feature's endpoints. No
   `createAsyncThunk` for I/O. No handwritten async middleware for API calls.

**Rationale.** RTK Query is the visible API boundary the middleware was approximating: endpoints
are named, cache and invalidation are built in, and token attach is one interceptor. Feature
`injectEndpoints` keeps I/O next to the feature without letting the feature own axios.
`axios-mock-adapter` makes that interceptor a Phase-1 code path.

**Alternatives.** Keep explicit middleware + a separate mock client (OQ-17) — rejected; header
interceptors would be dead until Phase 3. `fetchBaseQuery` without axios — rejected; the team
wants axios interceptors. Auth token only in RTK Query cache — rejected; we persist the session
in Keychain and must not persist the catalog cache. `createAsyncThunk` per operation — rejected;
it is the pattern doc 03 already forbade.

**Reversibility.** Medium for RTK Query (every endpoint is a hook). Low for axios (still
confined to `src/api/client/`). High for dropping the mock adapter in Phase 3.

## Consequences

- ADR-0002 point 3 ("features reach the API only through Redux middleware") is amended: they
  reach it through RTK Query hooks on `baseApi`. Axios stays.
- ADR-0017 stands: session-clearing 401 is still not in the raw interceptor. Its home is
  `baseQueryWithAuth`.
- Doc 04 drops the user and content slices; pagination hooks wrap RTK Query queries.
- Doc 12 tests the interceptor + `baseQueryWithAuth` + mock handlers, not a parallel mock
  client.
- DF1 still forbids axios/`fetch` from screens; the allowed I/O is generated hooks.
