# Doc 11 — Interface Contracts

**Version:** v0.4.1
**Status:** Draft
**Coverage:** full for Phase 1. The promotion to a machine-readable artifact (OpenAPI) is
consciously deferred with a named trigger (§2.3, ADR-0016) rather than left unenumerated.
**Last updated:** 2026-08-18 (STEP-3.5)
**Audience:** Mobile developers, QA, backend team (Phase 3)

> The one boundary in this system that will ever cross a network — five operations, their
> payloads, their errors — specified tightly enough that the mocks, the tests, and a future
> backend team all build against the same thing.

## Table of Contents

1. [Boundary contract inventory](#1-boundary-contract-inventory)
2. [Authoring source and contract of record](#2-authoring-source-and-contract-of-record)
3. [Artifact locations](#3-artifact-locations)
4. [Versioning & compatibility](#4-versioning--compatibility)
5. [The five operations](#5-the-five-operations)
6. [Request / message conventions](#6-request--message-conventions)
7. [Payload shapes](#7-payload-shapes)
8. [Error model](#8-error-model)
9. [Auth, authorization & privacy](#9-auth-authorization--privacy)
10. [Observability hooks](#10-observability-hooks)
11. [Contract testing & CI inputs](#11-contract-testing--ci-inputs)
12. [Ownership & review](#12-ownership--review)
13. [Deferred / informal interfaces](#13-deferred--informal-interfaces)
14. [Phase-3 contract checklist](#14-phase-3-contract-checklist)

---

## 1. Boundary contract inventory

Doc 03 §5 already established that most handoffs in this system are **in-process imports**, not
network APIs. This session's first job was to enumerate every boundary honestly — including the
ones that turn out not to need a contract — so a later reader can see what was considered rather
than what was merely remembered.

| # | Boundary | Kind | Owner | Contract level | Style | Status |
|---|----------|------|-------|----------------|-------|--------|
| **B-A** | Features → **API module** (via RTK Query hooks on `baseApi`) | Async, wire-shaped | API module | **Formal** | TS interfaces + §5–§8 tables | **This doc** |
| **B-B** | API module → future HTTP backend | Async, over the network | API module (consumer: backend team) | **Formal — same contract as B-A** | Same | Phase 3; §6.5 transport profile |
| **B-C** | Shell → Auth / Storefront / Shared / API | In-process module exports | App shell | Informal | Public exports + import rules | Doc 03 §8 |
| **B-D** | Auth / Storefront → Shared kernel | In-process component/token API | Shared kernel | Informal | Public component + token API | Doc 03 §8, doc 07 |
| **B-E** | App ↔ OS secure storage (persisted auth blob) | Library API + our blob shape | Auth feature | **Informal — owned elsewhere** | `version: 1`, purge on mismatch | Doc 04 §7, ADR-0003 |
| **B-F** | Build-time `@env` config module | Config surface | Whoever adds a key | **Informal — owned elsewhere** | `.env.example` is the contract | Doc 09 §4.2, §7.2 |

**Third-party APIs consumed:** none. No IdP, no analytics vendor, no crash reporter, no CDN, no
payments (docs 03 §7, 10 §5.1, 15 §4). This project consumes zero external contracts, which is
unusual and worth stating rather than leaving as an absence.

**Not boundaries at all in Phase 1:** webhooks, CLI interfaces, a published npm package, data
import/export formats, a queue or event bus, and service-to-service auth (doc 16 §7). None of
these have a referent in a single-process React Native app with in-process mocks.

### 1.1 B-A and B-B are one contract with two transports

These are not two contracts that happen to look alike. ADR-0002's entire purpose is that the wire
shape survives the mock → HTTP swap, so **the shapes are stated once** in §5–§8, and §6.5 records
the small set of things that genuinely differ between the two implementations of that one
contract. Two documents would guarantee drift — which is the specific failure this doc exists to
prevent.

### 1.2 Why B-E and B-F are listed but not specified here

Both are real boundaries and both already have a governing rule elsewhere:

- **B-E** — doc 04 §7 and ADR-0003: redux-persist `version: 1` on the auth slice; an incompatible
  persisted shape purges rather than migrates.
- **B-F** — doc 09 §4.2 and §7.2: a single gitignored `.env`, three keys, and `.env.example`
  updated in the same PR as any code that reads a new key.

Restating either here would create a second place that has to be kept true. A row plus a pointer
costs nothing and loses nothing. The one connection worth making explicit is directional:
**`API_BASE_URL` (B-F) is the field the transport profile in §6.5 flips at Phase 3** — the config
key and the contract are the same decision seen from two sides.

## 2. Authoring source and contract of record

### 2.1 Today

| Role | Where |
|------|-------|
| **Authoring source of truth** | **`src/api/types/`** in `quasar-disney-mobile-app` — `card.ts`, `container.ts`, `envelope.ts`, `auth.ts`, `errors.ts`, `index.ts`. Doc 04 §7's "types in the API module *are* the schema", now an actual file tree (STEP-3.1) |
| **Consumer-facing contract of record** | **This document.** §5–§8's tables are what a backend team is handed — not a link to a source file in a repo they do not have |

### 2.2 The inversion — stated on purpose, resolved in STEP-3.1

**Resolved in STEP-3.1.** The inversion this section existed to name is over: `src/api/types/`
in `quasar-disney-mobile-app` now transcribes §7 and §8 completely, so the authoring source and
the contract of record both exist and are in agreement. §3.1's obligation is discharged — the
types were written from this specification rather than from memory of a conversation, which is
the whole reason the obligation was stated rather than assumed.

The handover rule, now in its second half:

> **This doc's tables were normative until the TS types existed. Since STEP-3.1 the TS types are
> normative** — `src/api/types/` is where a wire shape is *decided* — and **this doc stays the
> consumer-facing contract of record**: §5–§8's tables are what a backend team is handed, not a
> link to a source file in a repo they do not have. Neither may move without the other:
> **§12's same-PR rule is what keeps them one contract instead of two**, and it binds every
> substep that touches an operation, payload, envelope, or error code.

Kept for the record, since it explains §3.1's shape: until STEP-2.2 the authoring source lived in
a repo that did not exist, and until STEP-3.1 that repo's `src/api/types/` was an empty folder.

### 2.3 The upgrade trigger

**Promotion to OpenAPI 3.1 is triggered by OQ-10** — the moment the backend team is actually
engaged. At that point the OpenAPI document becomes the contract of record and this doc becomes
the narrative around it. See **ADR-0016** for why the artifact is not written now, including the
argument against that call.

## 3. Artifact locations

| Artifact | Where, now | Where, after the rest of STEP-3 |
|----------|-----------|-------------------------------|
| Operation spec — paths, payloads, envelope, errors, transport profiles | **`architecture/11-interface-contracts.md`** (this doc) | Unchanged — remains the contract of record until OQ-10 |
| TS wire types (`Card`, `Container`, envelopes, `ApiError`, login / `/me` DTOs) | **`Code/quasar-disney-mobile-app/src/api/types/`** — the authoring source (STEP-3.1). `card.ts` §7.1 · `container.ts` §7.2 · `envelope.ts` §6.1–§6.3 · `auth.ts` §7.4 · `errors.ts` §8 · `index.ts` barrel | Unchanged — features import the barrel and nothing deeper in `src/api/` (doc 03 §8.2) |
| axios instance + interceptors + `axiosBaseQuery` / `baseQueryWithAuth` | Folder `src/api/client/` exists empty (STEP-2.2) | `src/api/client/` |
| RTK Query `baseApi` | Stub file `src/api/baseApi.ts` (STEP-2.2); `createApi` not called | `src/api/baseApi.ts`; feature `injectEndpoints` in `src/features/*/api.ts` |
| Mock fixtures + `axios-mock-adapter` | Folder `src/api/mocks/` exists empty (STEP-2.2) | `src/api/mocks/` — on the **same** axios instance (ADR-0020) |
| Contract tests | Do not exist | App repo, colocated `*.test.ts` beside the source (doc 12 §10.1). Test factories live in `src/api/mocks/` alongside the demo fixtures (doc 12 §4.1) |
| OpenAPI document | Does not exist by decision (ADR-0016) | `contracts/openapi.yaml` in the app repo, **at OQ-10 only** |

### 3.1 Scaffold STEP obligation

The application-repo scaffold is split across two STEPs (planning session):

**STEP-2** **must**:

1. Create the `src/api/` tree — including empty `src/api/types/`, `src/api/mocks/`,
   `src/api/client/`, and a stub `src/api/baseApi.ts`. Do **not** transcribe §7 here.
2. Add a line to the app repo's README pointing at this doc as the contract of record
   until STEP-3 types exist.
3. Update `registries/repos.yml` with the new repo (doc 03's standing rule).

**STEP-3** **must** transcribe §7's tables into TypeScript interfaces and enums in
`src/api/types/`, then stand up interceptors, `baseApi`, and mocks. From that PR onward
the TS types are normative (§2.2).

**Status:** the transcription landed in **STEP-3.1** — §7 and §8 are in `src/api/types/`, the
app README carries the contract-of-record line, and §2.2's handover is discharged. The
interceptors, `baseApi`, and mocks remain STEP-3.2–3.4.

This is the handoff that makes §2.2's inversion safe. Without it, the TS types get written from
memory of a conversation instead of from a specification.

### 3.2 Source layout — owned by doc 03

The folder layout above (`src/api/…`) is **component structure, not an interface contract**, so it
is recorded in **doc 03 §8** and merely referenced here. This doc owns *what crosses* the
boundary; doc 03 owns *where the code sits*.

## 4. Versioning & compatibility

Phase 1 has one client and one implementer, so the standard machinery — deprecation windows,
sunset headers, consumer migration schedules — has nobody to serve. What does have a referent is
what happens when a backend arrives and returns something we did not expect.

### 4.1 No path version segment

**No `/v1/`.** Adding one now guesses at a scheme the backend team has not agreed to (OQ-10), and
`API_BASE_URL` already absorbs it: if Phase 3's host wants a prefix, it goes in the env *value*,
not in the client's path constants.

Recorded honestly: a `/v1` prefix is nearly free and genuinely hard to retrofit into a *live*
client. This project does not have a live client — it has a demo behind a swappable adapter, and
the prefix is a one-line change to a base URL. Revisit at OQ-10 (§14).

### 4.2 The contract version is this doc's Version Log

Semver-ish, on the document — not on the wire. That is what a backend team is handed, and what a
breaking change bumps.

### 4.3 Compatibility rules

| Change | Breaking? | What it requires |
|--------|-----------|------------------|
| Adding an **optional** field | No | Nothing beyond the usual PR (doc 04 §7 already says this) |
| Adding an enum **value** | No — see §4.4 | A Version Log entry |
| Rename, removal, type change, enum-value **removal** | **Yes** | Version Log entry + TS types + mocks + tests updated in the same PR (§12) |

**Breaking changes are allowed freely until the backend team accepts the contract (OQ-10).** After
that they become a joint decision. This is stated explicitly so that a Phase-1 rename is not
mistaken for a process violation.

### 4.4 Unknown enum values — the one real forward-compat decision

`Container.variant` is a closed enum today (§7.2), and doc 04 explicitly parks `'live'` for a
later feature. The client **will** eventually meet a variant it does not know.

> **An unrecognized `variant` renders nothing and emits a `console.warn`.** It does not throw, and
> it does not fall back to a default layout.

Not throwing is what keeps a server-side variant addition non-breaking (§4.3). Not defaulting is
the more important half: a fallback layout would render a row *incorrectly but plausibly*, which
on a visual-fidelity demo is worse than rendering nothing at all.

## 5. The five operations

Doc 15 §8 paginates on **two axes** — vertical (more rows) and horizontal (more tiles in a row).
That second axis has never been written down as an operation. It is one, and it is why this table
has five rows rather than the four the earlier docs imply.

| # | Operation | Auth | Purpose |
|---|-----------|------|---------|
| 1 | `POST /auth/login` | — | Exchange credentials for a session |
| 2 | `GET /me` | Bearer | Hydrate the `getMe` cache |
| 3 | `GET /home-feed?cursor=&limit=` | Bearer | HomeFeed page — **vertical** paging (containers) |
| 4 | `GET /continue-watching?cursor=&limit=` | Bearer | Continue Watching page |
| 5 | `GET /containers/{containerId}/resources?cursor=&limit=` | Bearer | **Horizontal** paging — more cards in one row (`useCarouselPage`, doc 15 §8) |

Operation 5 is deliberately **flat** rather than nested under a feed (`/home-feed/containers/…`):
a row is addressable regardless of which feed it arrived in, and nesting would force Continue
Watching rows to need a second path for the identical operation.

## 6. Request / message conventions

### 6.1 Envelope — never a bare top-level array

Every paginated response uses the same shape:

```json
{
  "data": [ … ],
  "nextCursor": "opaque-string-or-null"
}
```

A bare top-level array cannot gain a field later without breaking every consumer. The envelope
costs one key.

### 6.2 Two cursors, two axes

This resolves doc 04 §1.4's open "container vs page envelope" question as **both, at different
levels** — they were never competing, they are different fields:

| Cursor | Lives on | Pages |
|--------|----------|-------|
| `nextCursor` on the **page envelope** | The response | The list of **containers** (vertical) |
| `nextCursor` on **each Container** | Each container object | That container's **`resources`** (horizontal) |

`null` means exhausted. Cursors are **opaque** (doc 04 §4) — the client never parses, compares, or
constructs one, and **always trusts `nextCursor` over arithmetic** on counts.

### 6.3 Field and value conventions

| Topic | Decision |
|-------|----------|
| **Casing** | `camelCase` throughout. There is no serialization layer between TypeScript and JSON; `snake_case` would buy a mapper and nothing else |
| **Content type** | `application/json; charset=utf-8` |
| **Timestamps** | **ISO 8601 UTC strings** — e.g. `"2026-08-24T10:15:00Z"`. See §6.4 |
| **IDs** | UUID v4 strings (doc 04 §4). No integers, no natural keys |
| **`limit`** | A request *hint*; the server may cap it. Defaults: **16** on `/home-feed` (the hero + 15 doc 04 locked), **10** on `/continue-watching` and on `resources` paging — this closes **OQ-23** |
| **Idempotency keys** | None. Login is the only non-GET and is naturally idempotent |
| **Filtering / sorting** | None. No query parameters beyond `cursor` and `limit` |
| **Partial updates** | N/A — no `PUT`, `PATCH`, or `DELETE` exists in Phase 1 |

### 6.4 Why `expiresAt` is an ISO string, not the JWT's numeric `exp`

Doc 04 §1.5 records `Session.expiresAt` as "Unix `exp`". On the **wire** it is an ISO 8601 string
instead, for two reasons:

1. It removes the seconds-vs-milliseconds ambiguity that numeric epochs reliably produce.
2. It means the client never decodes the JWT to learn when it expires — so no `jwt-decode`
   dependency enters doc 03 §7's hard-dependency list to satisfy a field the server could just
   send.

The auth slice may store it however it likes; this constrains the wire only.

### 6.5 Transport profiles

One contract, two implementations. Everything not in this table is identical.

| Aspect | Mock (Phase 1) | HTTP (Phase 3) |
|--------|----------------|----------------|
| Mechanism | `axios-mock-adapter` on the **same** axios instance RTK Query uses | axios over HTTPS via the same instance |
| Latency | Artificial **400–600 ms** (doc 05) | Real |
| Status codes | Synthetic HTTP status on the mocked response | Real HTTP status |
| `Authorization` header | Real header on the axios config, attached by the **request interceptor**; mock handlers validate it (§9.2) | Same interceptor |
| Base URL | `API_BASE_URL` inert (doc 09 §4.2) | `API_BASE_URL` addresses a real host |
| TLS / ATS | N/A | HTTPS only, ATS on, no cleartext exception (doc 06 §5) |
| Token expiry | Mock handlers mint and honour a 7-day `exp` (doc 04) | Server-side |
| Correlation ID | None (§10) | Joint decision at OQ-10 (§14) |
| Failure injection | Test-time seam only, never a runtime toggle (doc 09 §6.2) | N/A |

## 7. Payload shapes

Normative until the TS types exist (§2.2). Optional fields are marked `?`.

### 7.1 Card — closes OQ-26

| Field | Type | Notes |
|-------|------|-------|
| `id` | `string` (UUID) | |
| `title` | `string` | Content name. Shown on the tap-alert; on CW and landscape rows |
| `artwork` | `Partial<Record<AspectRatio, string>>` | Keys `'2:3'`, `'16:9'`, `'3:4'`; only the ratios a variant needs |
| `rating` | `string \| null` | Chip copy: `7+`, `13+`, `16+`, `ATP`, … |
| `releaseYear` | `number \| null` | Hero metadata |
| `genre` | `string \| null` | Hero metadata |
| `badge` | `string \| null` | Hero pill — placeholder copy, **not** a Disney mark |
| `tagline` | `string \| null` | Hero overlay; `null` on other variants |
| `progress?` | `number` | `0..1` fill for the cyan bar. Populated **only** inside a `progress` container |
| `remainingMinutes?` | `number` | Formatted via i18n — never a hardcoded phrase (doc 04 §1.3). `progress` only |
| `episodeLine?` | `string \| null` | e.g. `T3:E12 …`; `null` for movies. `progress` only |

`AspectRatio` = `'2:3' | '16:9' | '3:4'`.

**`title`, not `name`** — closing the half of OQ-22 doc 04 §1.2 left open. It is the domain term
for a piece of content (doc 04 v0.1 called the entity "Title"), and pairing it with
`Container.name` makes any payload self-describing at a glance: rows have names, content has
titles. Doc 04 §1.2 is updated to match.

### 7.2 Container

| Field | Type | Notes |
|-------|------|-------|
| `id` | `string` (UUID) | Row identity; the path parameter of operation 5 |
| `name` | `string` | Row header. **Display string, rendered verbatim** — see §7.3 |
| `variant` | `ContainerVariant` | `'hero' \| 'progress' \| 'standardPortrait' \| 'standardLandscape'`. Unknown values → §4.4 |
| `resources` | `Card[]` | One page of cards. **`resources`, not `items`** (ADR-0007) |
| `nextCursor` | `string \| null` | Horizontal cursor for this row's `resources` (§6.2) |

`'progress'` is **never** a member of a `/home-feed` response — it arrives only from
`/continue-watching` (ADR-0006). `'live'` is out until that feature exists. **`visibleCount`
is not a field** — tile peek is client-side, keyed by `variant` (doc 07, OQ-37 closed).

### 7.3 Two kinds of server-supplied string, and why they are treated oppositely

This is the one place the contract looks internally inconsistent, so it is stated directly rather
than left to be discovered:

| String | Rendered? | Why |
|--------|-----------|-----|
| `Container.name` | **Verbatim** | It is *data* — a row header the client cannot meaningfully translate. The server must return it localized to the request's locale (doc 04 §1.4, **OQ-30**) |
| `error.message` (§8) | **Never** | It is *developer-facing*. User-visible error copy is i18n keyed by `error.code` |

The rule underneath both: the client renders server strings that are **content**, and never
renders server strings that are **diagnostics**.

### 7.4 Operation payloads

**1 — `POST /auth/login`**

```json
// request
{ "email": "demo@example.com", "password": "…" }

// 200
{ "accessToken": "<jwt>", "expiresAt": "2026-08-24T10:15:00Z" }
```

**2 — `GET /me`**

```json
{ "id": "3f2a…", "userName": "Andrés" }
```

No email, no roles, not even a stub (doc 16 §4, §6).

**3 / 4 — `GET /home-feed`, `GET /continue-watching`**

```json
{ "data": [ /* Container */ ], "nextCursor": null }
```

HomeFeed first page: one `variant: "hero"` container plus 15 others. Further vertical pages
contain no second hero. Continue Watching: typically one container, `variant: "progress"`.

**5 — `GET /containers/{containerId}/resources`**

```json
{ "data": [ /* Card */ ], "nextCursor": "…" }
```

## 8. Error model

### 8.1 Not RFC 9457

Problem details was considered and declined for two concrete reasons, not for weight:

1. Its `type` member is a URI expected to dereference to an explanation. **Doc 08 §1 hosts
   nothing** — there is no page for it to point at.
2. Its `title` and `detail` are prose fields *designed to be displayed to a human*, which collides
   directly with doc 07 §9's i18n posture (Spanish is the reference locale). Rendering server
   English on stage would be a visible fidelity break.

Recorded as considered-and-declined with a Phase-3 revisit (§14) — a real backend team may
reasonably want it, and the mapping from §8.2 is mechanical.

### 8.2 The shape

```json
{ "error": { "code": "INVALID_CREDENTIALS", "message": "developer-facing, never rendered" } }
```

> **`code` is a closed enum the client switches on. `message` is developer-facing and is never
> rendered to a user.** All user-visible error copy is i18n keyed by `code`.

| `code` | Status | Raised by | Client behavior |
|--------|--------|-----------|-----------------|
| `INVALID_CREDENTIALS` | 401 | Operation 1 | Inline error on the credentials screen (doc 02 F2) |
| `UNAUTHORIZED` | 401 | Operations 2–5 | Clear session → Welcome |
| `VALIDATION_FAILED` | 400 | Any | Unused in Phase 1 (the client validates first). Optional `fields[]` |
| `NOT_FOUND` | 404 | Operation 5, unknown container | Storefront error state |
| `INTERNAL` | 500 | Any | Storefront error state / generic inline |

### 8.3 One `ApiError` across both transports

The mock has no real network, so the API module still normalizes both transports into a single
client-facing type (response interceptor maps HTTP → this shape):

```ts
ApiError { code: ErrorCode; status: number; message: string }
```

Mocks supply a synthetic `status` on the axios response. **Features and hooks never see an axios error or a raw
rejection** — which is precisely what makes the Phase-3 swap invisible above the boundary
(ADR-0002 / ADR-0020).

### 8.4 The 401 collision — a trap worth naming

A login failure and an expired token are both `401`. The "clear session → Welcome" reaction must
fire only on the second.

> **The session-clearing 401 policy is transport-agnostic and lives *above* the raw interceptor** — in
> **`baseQueryWithAuth`** (ADR-0020 / ADR-0017), which consumes the normalized `ApiError` (§8.3). It switches
> on **`code`**: `UNAUTHORIZED` clears the session and `resetApiState()`, `INVALID_CREDENTIALS` does not. It never compares
> paths.

If it fires on a login failure, a wrong password bounces the user out of the credentials screen
and **F2 — the inline error state, half of what Phase 1a exists to demonstrate — never renders**.
Distinct codes (`INVALID_CREDENTIALS` vs `UNAUTHORIZED`) make the distinction mechanical rather
than a path comparison.

**What the caller observes on `UNAUTHORIZED`** (measured in STEP-3.5's T2 suite). The base query
returns the `ApiError` in every case, but `resetApiState()` removes the in-flight query's cache
entry, which aborts its RTK Query thunk — so a caller awaiting that query sees it torn down rather
than errored. That is the intended shape of this reaction, not a gap: the session is over and the
user is on their way back to Welcome, so there is no screen left to render an error on. It is
recorded because the obvious test — "assert the hook surfaces `UNAUTHORIZED`" — cannot be written
against RTK Query, and the next person to try it should not conclude the policy is broken. What
*is* observable, and what the suite asserts, is the transport's normalized `ApiError` plus the
`sessionCleared` + `resetApiState()` pair that only this branch dispatches. `INVALID_CREDENTIALS`
is unaffected: nothing is reset, so **F2's error reaches the credentials screen normally.**

**Why not the axios interceptor** (**ADR-0017**). A global interceptor 401 handler would also fire on
login failure and destroy F2. The request interceptor attaches `Authorization`; the response
interceptor maps status → `code`; **`baseQueryWithAuth` reacts**. Phase 1 mocks use
`axios-mock-adapter` on the same instance, so this path runs in 1a (doc 12 §3.1).

### 8.5 No user enumeration

"No such email" and "wrong password" both return `INVALID_CREDENTIALS`. This is moot in Phase 1 —
there is exactly one demo account (RISK-0008) — but this project is a **migration template**
(doc 01), and a split error here is exactly the kind of thing that gets copied forward into a
product with real accounts.

### 8.6 Never in an error payload

Stack traces, internal paths, upstream vendor messages, SQL, or anything echoing the submitted
password. Per doc 10 §2.3, error *codes and messages* are fine to log; response *bodies* are not.

## 9. Auth, authorization & privacy

### 9.1 At the boundary

| Topic | Decision | Source |
|-------|----------|--------|
| Header | `Authorization: Bearer <jwt>`, attached by the axios interceptor | Doc 16 §6 |
| Authenticated operations | 2, 3, 4, 5. Operation 1 is not | §5 |
| Scopes / roles / permissions | **None.** Binary session gate — no `role` or `entitlements` field, not even a stub | ADR-0010, doc 16 §4 |
| Tenancy | **None.** No `X-Tenant-Id` | Doc 16 §5 |
| Deletion / export endpoints | **None.** No data subjects exist; privacy session stays Deferred | Doc 04 §5 |
| Audit-sensitive operations | **None** | Doc 06 |

A stub `role` field was specifically rejected: doc 16 §4 is explicit that Phase 3 adds claims *in
the contract*, and a placeholder is a shape the mock would have to lie about.

### 9.2 The mock handlers validate the token

> The `axios-mock-adapter` handlers **check the Bearer token's presence and `exp`** on operations 2–5 and return
> `UNAUTHORIZED` otherwise. They do not accept an arbitrary value. The request interceptor has already attached the header.

Doc 16 §4 implies this ("401 without a valid token") but nothing stated it as a contract
obligation. It matters because a permissive mock would leave doc 04's 7-day expiry path and the
`/me` 401 → Welcome flow **untested until Phase 3** — the exact branch that would then fail on
first contact with a real backend.

### 9.3 Personal data on the wire

| Data | Operation | Handling |
|------|-----------|----------|
| `email`, `password` | 1 (request) | Never stored, never logged (doc 04 §5) |
| `userName` | 2 (response) | PII-shaped fixture; never logged |
| `progress`, `remainingMinutes` | 4 | Confidential in the real product; fixture in Phase 1 |

All covered by doc 10 §2.3's never-log list — and its no-bodies rule means none of it can reach
the console by accident.

## 10. Observability hooks

**No correlation ID, no request ID, no trace headers.** Doc 10 §2.4's decision stands, and this
session deliberately does **not** reserve a header name: doc 10's reasoning is that the ID must be
minted in the interceptor *and* echoed by the backend, which makes it a joint decision rather than
one to guess at. It is item 2 on §14's checklist.

**Boundary logging.** The API module may log operation name, `status`, `code`, and duration.
Never request or response bodies, never the `Authorization` value (doc 10 §2.3).

**Analytics stub.** Unchanged (doc 10 §8): no PII in stub payloads.

## 11. Contract testing & CI inputs

Inputs for session 1.12, which owns the Test Strategy doc.

### 11.1 No consumer-driven contract testing

Pact and its relatives exist to coordinate teams that deploy independently. Here one process
contains both sides of the boundary. Revisit at OQ-10.

### 11.2 `tsc` is the primary contract test — conditionally

The type-as-contract strategy (§2.1) only holds if the fixtures are actually typed:

> **Mock fixtures are declared `Container[]` / `Card[]` — never `any`, and never untyped JSON
> imports.**

A loosely imported `resources.json` would make the entire strategy decorative: the types would
describe an intent nothing checks. This is the highest-leverage line in the section.

### 11.3 Behavioral tests 1.12 should specify

All unit-level, against the mock adapter and the middleware — which doc 02 already requires
("tests for every reducer/middleware/hook"). **1.12 is done:** every row below is on the must-cover
list in `architecture/12-test-strategy.md` §3, mapped to its tier (the 401-scoping and cursor rows
are integration-tier, since they need a real store plus middleware plus adapter wired together):

| Test | Guards |
|------|--------|
| `nextCursor` exhaustion on **both** axes | §6.2 |
| Unknown `variant` drops the row and warns; does not throw, does not default | §4.4 |
| **401 scoping** — login failure does *not* clear the session; `/me` 401 does | §8.4 |
| Expired mock JWT → `UNAUTHORIZED` | §9.2 |
| Both transports normalize to one `ApiError` | §8.3 |
| Injectable failure is reachable from tests only | Doc 09 §6.2 |

### 11.4 CI gates

**Amended in 1.12 — see ADR-0018.** This section previously read "none until Bitrise" (Phase 2,
OQ-05). That conflated two blockers: Bitrise is blocked on **accounts and signing** because it builds
native binaries, but `tsc --noEmit` and the Jest suite are plain Node and need no simulator, no
certificate, and — because tests resolve `@env` to a committed stub (doc 12 §4.3) — no secrets.

**CI is therefore two tiers:**

| Tier | Runner | What | Phase |
|------|--------|------|-------|
| **A — JS gate** | GitHub Actions, `ubuntu-latest`, every push + PR | `tsc --noEmit` · `jest` · `eslint` · `prettier --check` | **1a** |
| **B — Native build** | Bitrise | Native build, installable artifacts, release smoke | **Phase 2** (OQ-05) |

The commands are unchanged; what changed is that a machine runs them at the merge. Doc 12 §7.1 is the
authoritative gate list, and it adds lint rules that make DF1, DF5/A5, and A2 mechanical.

**Still no OpenAPI/GraphQL/protobuf linting** — there is no such artifact by decision (ADR-0016), and
adding a linter for a file that does not exist would be theatre. This tiering **adds no environment**:
doc 09 decision 3 holds, CI is a runner.

## 12. Ownership & review

**Owner:** the mobile pair owns this contract outright until OQ-10. After the backend team accepts
it, breaking changes are a joint decision. **OQ-12 closed:** Dev A = Raul Angel, Dev B = Andres Montoya.

**The update rule** — a definition-of-done item on any implementation substep that touches an
operation, payload, envelope, or error code:

> This doc's tables, the TS types, the mocks, the tests, and the app repo README link are updated
> **in the same PR**. A wire change that lands without them is an incomplete change, reviewable as
> such.

This is the third time this project has reached for the same discipline — doc 09 §7.2 applies it
to `.env.example`, doc 03's standing rule applies it to `registries/repos.yml`. Naming it once:
**an artifact that describes code is updated in the PR that changes the code, or it is not a
description, it is a rumour.**

## 13. Deferred / informal interfaces

| Interface | Disposition | Revisit trigger |
|-----------|-------------|-----------------|
| B-C, B-D (in-process module seams) | **Informal by decision.** Governed by doc 03 §8's import rules | A module is genuinely extracted to a package (doc 02 DF5) |
| B-E (persisted auth blob) | **Owned elsewhere** — doc 04 §7, ADR-0003 | A second persisted slice appears |
| B-F (`@env` config) | **Owned elsewhere** — doc 09 §4.2, §7.2 | A fourth key, or per-configuration values (OQ-31) |
| OpenAPI artifact | **Deferred** — ADR-0016 | **OQ-10** |
| RFC 9457 problem details | **Declined** — §8.1 | OQ-10 |
| `/v1` path prefix | **Declined** — §4.1 | OQ-10 |
| Correlation ID | **Declined** — doc 10 §2.4 | OQ-10 / Phase 3 |
| Consumer-driven contract tests | **Declined** — §11.1 | OQ-10 |
| Third-party API contracts | **None exist** | A vendor is adopted (none planned before Phase 3) |

## 14. Phase-3 contract checklist

Eight decisions across six documents are parked on the same trigger — **OQ-10, the moment the
backend team is engaged**. Collected here so it is one page to walk rather than an archaeology
exercise.

| # | Item | Parked in |
|---|------|-----------|
| 1 | Promote to **OpenAPI 3.1**; it becomes the contract of record | §2.3, ADR-0016 |
| 2 | **Correlation-ID header** — name, who mints it, echo behavior | Doc 10 §2.4 |
| 3 | **Real JWT claims** and the managed IdP vendor | OQ-03, doc 16 §2 |
| 4 | **RFC 9457** — adopt, or keep the `{ error: { code, message } }` envelope | §8.1 |
| 5 | **`/v1` prefix** — in the path, or absorbed by `API_BASE_URL` | §4.1 |
| 6 | **Reachability probe** once a real host exists | OQ-29, doc 15 §2 |
| 7 | **Rate limiting / brute-force** posture on the login endpoint | RISK-0009, doc 06 §6 |
| 8 | **Server-localized `Container.name`** per the request's locale | OQ-30, doc 07 §9 |

---

## Decision Summary

| # | Decision | Choice | Rationale | Forecloses / tradeoff |
|---|----------|--------|-----------|-----------------------|
| 1 | Boundary inventory | Six boundaries enumerated; **one formal** (B-A/B). No third-party contracts consumed | Doc 03 §5 already found most seams are in-process imports | Formal contracts between in-process modules |
| 2 | B-A and B-B | **One contract, two transport profiles** (§6.5) — not two documents | ADR-0002's whole point is that the shape survives the swap; two docs guarantee drift | Documenting mock and HTTP semantics separately |
| 3 | B-E, B-F | **Listed, not specified** — cross-referenced to docs 04/09 | A second place to keep true is the drift risk this doc exists to prevent | Doc 11 as a single index of every rule |
| 4 | Contract level | Formal-but-lightweight for B-A/B; informal elsewhere | One consumer, one producer, one process today | Ceremony with no second party to serve |
| 5 | Style | **TS interfaces + this doc's Markdown tables**; no OpenAPI yet (**ADR-0016**) | An OpenAPI file with no server behind it rots between now and Phase 3 | A codegen-ready artifact for the backend team today — the migration-template argument against this is recorded in ADR-0016 |
| 6 | Source of truth | **Authoring:** TS types. **Contract of record:** this doc. Inverts when STEP-3 transcribes §7 | The app repo `quasar-disney-mobile-app` exists at `Code/quasar-disney-mobile-app/`; `src/api/types/` is empty until STEP-3 | Requires §3.1's split obligation (tree in STEP-2, types in STEP-3) |
| 7 | Versioning | **No `/v1`**; the contract version is this doc's Version Log | `API_BASE_URL` absorbs a prefix; the scheme is the backend team's to pick | Retrofitting a prefix into a live client — not a situation this project has |
| 8 | Compatibility | Additive-optional non-breaking; renames/removals breaking. **Free to break until OQ-10** | One consumer, in the same repo | Treating a Phase-1 rename as a process violation |
| 9 | Unknown enum values | **Drop the row + `console.warn`.** Do not throw, do not default | Not throwing keeps variant additions non-breaking; not defaulting avoids a plausible-but-wrong row on a fidelity demo | A graceful fallback layout |
| 10 | Operation count | **Five** — horizontal `resources` paging was never written down | Doc 15 §8 paginates on two axes | Pretending the carousel page is not an API call |
| 11 | Envelope | `{ data, nextCursor }` — **never a bare top-level array** | A bare array cannot gain a field without breaking every consumer | One key of overhead |
| 12 | Cursors | **Two**, at different levels — page envelope pages containers, each Container pages its `resources` | Closes doc 04 §1.4's "container vs envelope" as *both* | A single cursor that cannot express both axes |
| 13 | Conventions | camelCase; ISO 8601 UTC; UUIDs; `limit` a hint; no idempotency keys, filtering, or partial updates | No serialization layer; five operations; one non-GET | snake_case + a mapper |
| 14 | `expiresAt` | **ISO string on the wire**, not the JWT's numeric `exp` | Kills the seconds-vs-ms bug class; no `jwt-decode` dependency enters doc 03 §7 | Echoing the claim verbatim |
| 15 | Page sizes | `/home-feed` **16**; `/continue-watching` and `resources` **10**. Closes **OQ-23** | Matches doc 04's hero + 15 | A negotiated page size |
| 16 | `Card.title` | **`title`**, not `name`. Closes **OQ-22** | Domain term; `Container.name` / `Card.title` makes payloads self-describing | Doc 04 v0.2.0's symmetry, and a generic `.name` reader |
| 17 | Card fields | Doc 04 §1.2's working set locked; §1.3 progress fields optional and `progress`-only. Closes **OQ-26** | The UI working set is already known from 1.7 | Speculative production fields |
| 18 | Error model | **Not RFC 9457.** `{ error: { code, message } }`; `code` is switched on, `message` is never rendered | 9457's `type` needs a host we do not have, and its prose fields fight i18n | A standard shape the backend team may prefer (§14 item 4) |
| 19 | 401 scoping | Session-clearing reaction **excludes login** — lives in **`baseQueryWithAuth`** (ADR-0017 / ADR-0020), switching on `code`, not in the axios interceptor | Otherwise a wrong password destroys the F2 inline-error demo | Following the common "handle 401 in the interceptor" idiom |
| 20 | User enumeration | One `INVALID_CREDENTIALS` for both failure modes | Moot today; this is a migration template | — |
| 21 | `ApiError` | Both transports normalize to one type; features never see an axios error | What makes the Phase-3 swap invisible above the boundary | — |
| 22 | Mock token validation | The mock handlers **validate presence + `exp`**, not just presence | Otherwise the expiry and `/me` 401 paths are untested until Phase 3 | A trivially permissive mock |
| 23 | Observability at the boundary | No correlation ID and **no reserved header name** | It is a joint decision with whoever echoes it (doc 10 §2.4) | Retrofit cost at Phase 3, knowingly accepted |
| 24 | Contract testing | **`tsc` is the contract test** — conditional on fixtures being typed, never `any` | Untyped fixtures make type-as-contract decorative | Runtime schema validation (Zod et al.) |
| 25 | CI gates | **Amended (1.12, ADR-0018): two tiers** — a JS gate (`tsc --noEmit` · `jest` · `eslint` · format) on GitHub Actions in 1a; Bitrise owns the native build in Phase 2. No schema linting | The JS suite needs no simulator, certificate, or secret (doc 12 §4.3); only the *native* build is blocked on signing | Adds a blocking gate during doc 02 §9's parallel window (**OQ-36**) |
| 26 | Update rule | Doc + types + mocks + tests + README **in the same PR** | Third use of this discipline in the project; named once here | — |
| 27 | Phase-3 checklist | **§14** collects eight deferrals parked on OQ-10 | One page to walk beats archaeology across six docs | Must be kept current as more items park there |

## Open Questions

| ID | Question | Owner | Feeds into |
|----|----------|-------|------------|
| OQ-34 | Does the backend team accept this contract as written, and which of §14's eight items do they want to change? | Backend team / Mobile | Phase 3 — the concrete form of **OQ-10** |

Carried forward, unchanged by this session: **OQ-03** (production JWT claims + IdP vendor —
§14 item 3), **OQ-10** (backend accepts the contract — now expressed concretely as §14 and OQ-34),
**OQ-29** (reachability probe — §14 item 6), **OQ-30** (server-localized
`Container.name` — §7.3, §14 item 8), **OQ-31** / **OQ-32** (Phase 2/3 config and CI), **OQ-33**
(error-boundary fallback design).

**Closed by the planning session:** **OQ-12** (Dev A = Raul Angel, Dev B = Andres Montoya), **OQ-18** (`quasar-disney-mobile-app`), **OQ-28** (Raul Angel owns the sign-off binary).

**Closed by this session (1.11):** **OQ-17**, **OQ-22**, **OQ-23**, **OQ-26**. **OQ-17 reversed in 1.14 (ADR-0020):** mocks are `axios-mock-adapter` on the real axios instance so interceptors run.

## Version Log

| Version | Date | STEP | Change |
|---------|------|------|--------|
| v0.1.0 | 2026-08-17 | STEP-1.11 | Initial draft from the interface-contracts session. Six boundaries inventoried, one formal. Five operations named (horizontal `resources` paging was previously undocumented). Envelope, two-axis cursors, conventions, payload shapes, and error model locked. ADR-0016. Closed OQ-17, OQ-22, OQ-23, OQ-26; opened OQ-34. Doc 03 §8 gains the source-layout table; doc 04 §1.2 renames `name` → `title`. |
| v0.2.0 | 2026-08-17 | STEP-1.12 | **§8.4 amended (ADR-0017):** the session-clearing 401 policy lives **above the transport** and switches on `code` — in the interceptor it would sit in a path Phase 1 never executes. **§11.4 amended (ADR-0018):** CI is two tiers, a JS gate in 1a plus Bitrise's native build in Phase 2, replacing "none until Bitrise". §3 contract-test row and §11.3 point at doc 12. Decision Summary rows 19 and 25 updated. No wire-shape change: no field, name, enum, envelope, or error code differs. |
| v0.3.0 | 2026-08-17 | STEP-1.14 | Transport is RTK Query `baseApi` + axios interceptors; Phase 1 mocks are `axios-mock-adapter` on the same instance (**ADR-0020**). §8.4 home is `baseQueryWithAuth`. Reversed OQ-17. No wire-shape change. |
| v0.3.1 | 2026-08-17 | STEP-1.14 | §7.2: `visibleCount` is not a wire field (OQ-37). |
| v0.3.2 | 2026-08-17 | planning session | Closed OQ-12 / OQ-18 / OQ-28 as recorded by the planning session. |
| v0.3.3 | 2026-08-18 | STEP-2.2 | §3.1 split: STEP-2 creates the `src/api/` tree + README pointer + `repos.yml`; STEP-3 transcribes §7. Repo exists; types still pending. No wire-shape change. |
| v0.4.0 | 2026-08-18 | STEP-3.1 | **§7 and §8 transcribed into TypeScript.** The authoring source of truth is now `src/api/types/` in `quasar-disney-mobile-app` (§2.1, §3); §2.2's inversion is **resolved** and the TS types are normative from here, with this doc remaining the consumer-facing contract of record under §12's same-PR rule. §3.1 records the obligation as discharged. **No wire-shape change:** no field, name, enum value, envelope shape, page-size default, or error code differs from §6.3, §7, or §8 as written. |
| v0.4.1 | 2026-08-18 | STEP-3.5 | §8.4 records what a caller observes when the `UNAUTHORIZED` branch fires: `resetApiState()` aborts the in-flight query, so the reaction supersedes the error result. Behavior unchanged and §11.3's table unchanged — this is the T2 suite writing down what it found so the next author does not read the absent error as a defect. No wire-shape change. |
