# API design standards — quasar-disney-mobile

> **Reconciled to this project in STEP-1.12.** The shipped default was a generic, opinionated REST
> house style, and `architecture/11-interface-contracts.md` deliberately went the other way on six of
> its rules — casing, error format, correlation IDs, versioning, idempotency keys, and schema
> linting — each with recorded reasoning. Left unamended, this file would have described a style the
> project's own contract violates. The reconciliation and its rationale are in
> `architecture/12-test-strategy.md` §10.2.
>
> **Doc 11 is the contract of record and it wins.** This file is thin by design: it holds the
> house-style rules that generalize past the five operations doc 11 specifies, plus the deliberate
> deviations from published guides recorded in §"Deviations" so nobody "fixes" them later. Where a
> rule here and doc 11 disagree, doc 11 is right and this file has drifted.

**Scope:** REST / HTTP+JSON. This project has exactly **one** contract-bearing boundary — the API
module's five operations (doc 11 §1, boundary B-A/B-B). In Phase 1 it is served by an in-process mock
adapter; in Phase 3 the same contract runs over HTTPS (doc 11 §6.5). No GraphQL, no gRPC, and **no
third-party API is consumed** — doc 11 §1 records zero external contracts.

**Baseline:** published guides (Google AIPs, Microsoft, Zalando) are reference material, not
authority. Where this project departs from them, it does so on the record — see §"Deviations".
**Spectral / OpenAPI linting does not apply:** no OpenAPI artifact exists by decision (**ADR-0016**),
and doc 11 §11.4 calls linting a file that does not exist theatre. The trigger for both the artifact
and its linting is doc 11 §14 item 1.

## Resource naming & URLs
- **Nouns, not verbs**; the HTTP method is the verb. `GET /containers/{id}/resources`, not
  `GET /getContainerResources`.
- **Plural collections**, lowercase, `kebab-case` for multi-word path segments: `/home-feed`,
  `/continue-watching` — not `/homeFeed` or `/home_feed`.
- Nest one level at most (`/containers/{id}/resources`); link by ID past the first level. Doc 11 §5
  records why operation 5 is deliberately **flat** rather than nested under a feed: a row is
  addressable regardless of which feed delivered it.
- Query parameters are for pagination only — `cursor` and `limit`. **No filtering or sorting
  parameters** exist (doc 11 §6.3).

## HTTP methods & status codes
- Methods keep their defined semantics: **GET** safe/read, **POST** create or non-idempotent action.
  `PUT`, `PATCH`, and `DELETE` **do not exist** in this API (doc 11 §6.3) — there are no partial
  updates and nothing to delete.
- Return meaningful status codes; never bury an error in a `200`. The codes this contract uses are
  fixed in doc 11 §8.2: `401`, `400`, `404`, `500`.
- **The mock transport supplies a synthetic `status`** on `ApiError` so both transports present the
  same surface (doc 11 §6.5, §8.3). A status code is part of the contract even where no HTTP layer
  exists.

## Field conventions
- **Casing: `camelCase`**, uniformly, on every field. Doc 11 §6.3 — there is no serialization layer
  between TypeScript and JSON, so `snake_case` would buy a mapper and nothing else. (This is the
  fork the shipped default resolved as `snake_case`; the alternative it names — a JS/TS-consumed API
  — is exactly this project.)
- **Timestamps: RFC 3339 / ISO 8601, UTC, `Z`-suffixed** — `2026-08-24T10:15:00Z`. Never epoch
  integers, never local time. Suffix `At` (`expiresAt`). Doc 11 §6.4 records why the login response
  sends `expiresAt` as an ISO string rather than the JWT's numeric `exp`: it removes the
  seconds-vs-milliseconds bug class and means the client never decodes the token.
- **IDs: UUID v4 strings** (doc 04 §4). No integers, no natural keys.
- **Booleans read as predicates** (`isActive`, `hasMore`).
- **Enum value casing splits by kind — deliberately:**
  - **Domain enum values follow field casing** — `camelCase`: `'hero'`, `'progress'`,
    `'standardPortrait'`, `'standardLandscape'` (doc 11 §7.2).
  - **Error codes are `UPPER_SNAKE_CASE`** — `INVALID_CREDENTIALS`, `UNAUTHORIZED` (doc 11 §8.2).

  Stated explicitly because it looks like an inconsistency and is not: without this rule the mock and
  a future backend will disagree on the first new value either one adds.
- **Unknown enum values must not break a client.** An unrecognized `Container.variant` **renders
  nothing and emits `console.warn`** — it does not throw and it does not fall back to a default
  layout (doc 11 §4.4). Not throwing keeps server-side enum additions non-breaking; not defaulting
  avoids rendering a row incorrectly-but-plausibly, which on a visual-fidelity demo is worse than
  rendering nothing.
- **`null` vs omitted — the rule, not a habit:** `null` for a field that **always exists but may be
  empty** (`rating`, `releaseYear`, `tagline`); **omitted** (`?`) for a field that **does not apply
  to that variant** (`progress?`, `remainingMinutes?`, `episodeLine?` — populated only inside a
  `progress` container). Doc 11 §7.1.
- **Money:** N/A — no payments at any planned phase.

## Pagination
- **Cursor-based, opaque.** The client never parses, compares, or constructs a cursor, and **always
  trusts `nextCursor` over arithmetic** on counts (doc 04 §4, doc 11 §6.2).
- **Body envelope, never a bare top-level array, and no `Link` header:**

  ```json
  { "data": [ … ], "nextCursor": "opaque-string-or-null" }
  ```

  A bare array cannot gain a field later without breaking every consumer; the envelope costs one key
  (doc 11 §6.1).
- **Two cursors at two levels** (doc 11 §6.2) — this is the part most likely to be got wrong by
  someone reading only this file:

  | Cursor | Lives on | Pages |
  |--------|----------|-------|
  | `nextCursor` on the page envelope | The response | The list of **containers** (vertical) |
  | `nextCursor` on each `Container` | Each container object | That row's **`resources`** (horizontal) |

  `null` means exhausted.
- **`limit` is a request hint the server may cap.** Defaults: **16** on `/home-feed` (the hero + 15
  of doc 04), **10** on `/continue-watching` and on `resources` paging (doc 11 §6.3). These are the
  project's numbers — not the generic "default 25, max 100".

## Errors
- **Not RFC 9457 Problem Details.** Declined on two concrete grounds (doc 11 §8.1): its `type`
  member is a URI expected to dereference to an explanation, and **doc 08 §1 hosts nothing** for it
  to point at; and its `title`/`detail` are prose fields designed to be shown to a human, which
  collides with doc 07 §9's i18n posture, where Spanish is the reference locale and rendering server
  English would be a visible fidelity break. Revisit at doc 11 §14 item 4.
- **The shape:**

  ```json
  { "error": { "code": "INVALID_CREDENTIALS", "message": "developer-facing, never rendered" } }
  ```

  **`code` is a closed enum the client switches on. `message` is developer-facing and is never
  rendered to a user** — all user-visible error copy is i18n keyed by `code` (doc 11 §8.2).
- **The general rule underneath:** the client renders server strings that are **content**
  (`Container.name`, verbatim) and never renders server strings that are **diagnostics**
  (`error.message`). Doc 11 §7.3.
- **Never leak internals** — no stack traces, internal paths, SQL, upstream vendor messages, or
  anything echoing a submitted password (doc 11 §8.6). Error *codes and messages* may be logged;
  response **bodies** may not (doc 10 §2.3).
- **No correlation or request ID, and no reserved header name.** Declined in doc 10 §2.4 and doc 11
  §10: the ID must be minted by the client *and* echoed by the backend, which makes it a joint
  decision rather than one to guess at. Doc 11 §14 item 2.
- **No user enumeration:** "no such email" and "wrong password" both return `INVALID_CREDENTIALS`
  (doc 11 §8.5). Moot with one demo account, and this project is a migration template.
- **Scope the session-clearing 401.** A login failure and an expired token are both `401`. The
  session-clearing reaction fires only on `UNAUTHORIZED`, never on `INVALID_CREDENTIALS`, and it
  lives **above the transport** — not in the axios interceptor (**ADR-0017**, doc 11 §8.4). Switch on
  `code`, never on a path comparison.

## Versioning
- **No `/v1/` path segment.** Doc 11 §4.1 — adding one now guesses at a scheme the backend team has
  not agreed to, and `API_BASE_URL` already absorbs a prefix if Phase 3 wants one. Recorded honestly
  there: a prefix is nearly free and hard to retrofit into a *live* client, which this project does
  not have. Revisit at doc 11 §14 item 5.
- **The contract version is doc 11's Version Log** — semver-ish, on the document, not on the wire.
- **Compatibility** (doc 11 §4.3): adding an optional field or a new enum *value* is non-breaking;
  renames, removals, type changes, and enum-value removals are breaking and require the Version Log,
  the TS types, the mocks, and the tests updated **in the same PR**. Breaking changes stay free until
  the backend team accepts the contract (**OQ-34**) — a Phase-1 rename is not a process violation.
- **Clients must ignore unknown fields.**

## Idempotency, safety & limits
- **No `Idempotency-Key`.** Doc 11 §6.3 — login is the only non-GET operation and is naturally
  idempotent, so there is no double-create or double-charge to protect against.
- **Rate limiting and `429` are deferred**, not declined: there is no host to limit. Recorded as
  **RISK-0009** with doc 11 §14 item 7 as the trigger. When a real login endpoint exists, brute-force
  posture is a real decision.
- **Authenticate every non-public operation.** Operations 2–5 carry `Authorization: Bearer <jwt>`,
  attached by the interceptor (doc 11 §9.1). Operation 1 does not. **The mock adapter validates the
  token's presence *and* `exp`** and returns `UNAUTHORIZED` otherwise (doc 11 §9.2) — a permissive
  mock would leave the expiry path untested until Phase 3.
- **No scopes, roles, entitlements, or tenancy** — not even a stub field (ADR-0010, doc 16 §4–§5). A
  placeholder is a shape the mock would have to lie about.
- **TLS:** N/A in Phase 1 (no wire). In Phase 3, **HTTPS only, ATS on, no cleartext exception**
  (doc 06 §5, doc 11 §6.5).

## Deviations from the published guides, on the record
Three places where this contract knowingly breaks a rule the guides state. Recorded so a future
reviewer does not "correct" them:

| Deviation | Rule broken | Why it stands |
|---|---|---|
| **`POST /auth/login`** is a verb path | Nouns, not verbs | Near-universal auth idiom; already in the contract handed to the backend team (criterion A4) |
| **`GET /me`** is a singleton, not a plural collection | Plural collections | Same — the canonical spelling of "the current user" across the industry |
| **Enum casing is not uniform** — `camelCase` domain values, `UPPER_SNAKE_CASE` error codes | "Enums are lowercase strings" | Two different kinds of enum: one is domain data that follows field casing, the other is a control code. See §"Field conventions" |
