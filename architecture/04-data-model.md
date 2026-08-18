# Doc 04 — Data Model, Ownership & Retention

**Version:** v0.2.6
**Status:** Draft
**Last updated:** 2026-08-17 (STEP-1.14)
**Audience:** Mobile developers, backend team, QA

> Entities the React Native client models, who owns each one, where it lives on device, how long it is kept, and which of it is sensitive — with no server database in this project.

## Table of Contents

1. [Entity model](#1-entity-model)
2. [Ownership](#2-ownership)
3. [Storage](#3-storage)
4. [Identifiers](#4-identifiers)
5. [PII / sensitive data, retention, and deletion](#5-pii--sensitive-data-retention-and-deletion)
6. [Fetch topology and composition](#6-fetch-topology-and-composition)
7. [Consistency and schema evolution](#7-consistency-and-schema-evolution)

---

## 1. Entity model

There is **no backend datastore** in this project. The nouns below are TypeScript models: mock fixtures today, the same shapes over HTTP in Phase 3.

```
User 1 ── 1 Session
Card
Container * ── * Card          (ordered membership via resources)
HomeFeed page          = Container[]     (first page: 1 hero + 15 others)
ContinueWatching page  = Container[]     (typically 1 container, variant progress)
```

**Client-composed home (not an API resource):**

```
[ HomeFeed hero container ] + [ CW progress container ] + [ remaining HomeFeed containers ]
```

Continue watching sits **directly under hero** and shifts the other HomeFeed rows down one. Storefront owns that merge (ADR-0006). Both GETs use the same **Container** / **Card** types (ADR-0007); CW is still a **separate JWT API**, not mixed into the HomeFeed response (doc 05).

### 1.1 Entities

| Entity | What it is | Phase 1 |
|--------|------------|---------|
| **User** | Identity whose profile the UI shows | `id` (JWT `sub`) + **`userName`**. No email in the slice. Hydrated by authenticated `/me` |
| **Session** | The logged-in token | JWT string + `exp`. Auth slice. The only persisted entity |
| **Card** | Catalog item a tile represents (was “Title” in v0.1) | Shared across container variants. Artwork is a map on Card, not its own entity |
| **Container** | A row: own name + metadata, variant, **`resources: Card[]`** | Config-driven variants stay config, not types. One component for all variants |
| **HomeFeed** | Paginated **`Container[]`** | First page: **one `hero` + 15 other containers**. Not a stored aggregate |
| **ContinueWatching** | Paginated **`Container[]`**, JWT-gated | Typically **one** container, `variant: "progress"`. Same type as HomeFeed; **not** in the HomeFeed response |

**Not entities:** Credentials (login request DTO), Artwork (URIs on Card), WatchProgress (optional fields on a `progress` Card — see §1.3), LiveBroadcast (deferred; later feature), Profile, Download, Search, Payment, pagination cursors (wire only).

### 1.2 Card (closes OQ-02; field set closed in 1.11)

**OQ-26 is closed (1.11):** the working set below **is** the Card, with §1.3's progress fields as
optional members populated only inside a `progress` container. Wire names are locked in
`architecture/11-interface-contracts.md` §7.1.

| Field | Type | Notes |
|-------|------|--------|
| `id` | UUID string | Public; tiles, later details, CW merge |
| `title` | string | Content name. Shown on tap-alert; CW and landscape rows. **Renamed from `name` in 1.11** (OQ-22): `Container.name` / `Card.title` keeps rows and content distinguishable in a payload |
| `artwork` | `Partial<Record<AspectRatio, uri>>` | Keys: `'2:3'` (portrait), `'16:9'` (`progress` / landscape), `'3:4'` (hero). Only ratios a variant needs |
| `rating` | string \| null | Chip copy: `7+`, `13+`, `16+`, `ATP`, … |
| `releaseYear` | number \| null | Hero metadata |
| `genre` | string \| null | Hero metadata |
| `badge` | string \| null | Hero pill (placeholder copy, not a Disney mark) |
| `tagline` | string \| null | Hero overlay; null on other variants |

`AspectRatio` is `'2:3' | '16:9' | '3:4'`. URIs point at bundled placeholder art (`architecture/assets/placeholder-art/`).

### 1.3 Continue Watching cards (`variant: "progress"`)

Not a separate table. CW **resources** are **Cards**. Timeline chrome is selected by the container variant **`progress`**. Phase 1 mocks may add:

| Field | Type | Notes |
|-------|------|--------|
| `progress` | number | `0..1` fill for the cyan bar |
| `remainingMinutes` | number | Format via i18n (e.g. “11 min restantes”); do not hardcode the phrase |
| `episodeLine` | string \| null | e.g. `T3:E12 …`; null for movies |

**Resolved (1.11, closing OQ-26):** these stay **on Card as optional fields**, populated only
inside a `progress` container (doc 11 §7.1). The Continue Watching **request sends the JWT**. The mock (and later the backend) keys progress off that session. There is no WatchProgress id.

### 1.4 Container (HomeFeed and Continue Watching)

Same type on **both** authenticated feed GETs (ADR-0007).

| Field | Type | Notes |
|-------|------|--------|
| `id` | UUID string | Row identity for paging cards |
| `name` | string | Row header. **Display string, rendered verbatim** — a feed-supplied name is data, not UI copy, so the client cannot translate it. Phase-1 mocks send Spanish; the real API must return text localized to the request's locale (doc 07 §9, OQ-30) |
| `variant` | enum | `'hero' \| 'progress' \| 'standardPortrait' \| 'standardLandscape'`. **`'progress'` is not a HomeFeed response member** — it comes from the CW GET. `'live'` is out until that feature lands. **An unrecognized value drops the row with a `console.warn`** (doc 11 §4.4) |
| `resources` | Card[] | One page of cards (not `items`) |
| `nextCursor` | string \| null | Opaque; Phase 1 mocks may send `null`. **Placement resolved (1.11): both** — the page envelope's cursor pages containers (vertical), this one pages the row's `resources` (horizontal). Doc 11 §6.2 |

**Not a wire field:** `visibleCount` (tiles per rail) is client-side, keyed by `variant` (OQ-37 closed 1.14).

HomeFeed first page: **`Container[]`** with one `variant: "hero"` plus **15** other containers. Further vertical pages, if any, are more containers only (no second hero). Phase 1 mocks may set HomeFeed `nextCursor` to `null`.

Continue Watching: **`Container[]`** (typically length 1), each with `variant: "progress"` and `resources: Card[]`.

### 1.5 User and Session

| Field | Type | Notes |
|-------|------|--------|
| User.`id` | UUID | JWT `sub`; not shown in UI |
| User.`userName` | string | The only profile field the UI shows. Comes from `/me`, not from login |
| Session.`accessToken` | string | Mock JWT (`sub` + `exp` + `iat`; doc 16) |
| Session.`expiresAt` | number | Unix `exp` **in the slice**. Mock TTL **7 days** from each successful login. **On the wire it is an ISO 8601 UTC string** (doc 11 §6.4) — so the client never decodes the JWT to learn its expiry |

Login body: `{ email, password }` — request DTO, never stored.

**`GET /me` body (OQ-25 closed):** `{ id, userName }`. No email, no roles. **Paths and JSON names locked in 1.11** (doc 11 §5, §7.4 — OQ-22 closed).

---

## 2. Ownership

| Entity | Authoritative owner | Who else reads it | Notes |
|--------|---------------------|-------------------|--------|
| **Session** | Auth feature (auth slice) | Shell (nav, theme, boot gate) | Persisted |
| **User** | Auth feature (`getMe` cache; chrome selectors) | Shell / chrome if needed; **not** Storefront | Memory only (RTK Query cache). Auth ↛ Storefront still holds |
| **Card, Container, HomeFeed** | API module (wire + fixtures) | Storefront RTK Query cache | Storefront never authors catalog |
| **ContinueWatching** | API module (JWT-authenticated fixtures) | Storefront RTK Query cache | Same Container/Card types; `progress` is on CW cards, not in Auth |
| **Composed home list** | Storefront feature | — | Inserts the `progress` container under hero (ADR-0006) |
| **Cold-start loading** | App shell | — | Blocks until `/me` + HomeFeed + CW complete |
| **Silent CW reload** | Storefront feature | — | When the storefront screen is shown again |

---

## 3. Storage

No relational database, no SQLite/MMKV, no mock HTTP server acting as a DB.

| Data | Where | Engine |
|------|--------|--------|
| **Session** (JWT + `exp`) | Device, encrypted key-value | redux-persist whitelist = **auth slice only** + `react-native-encrypted-storage` |
| **User** | Memory only | `getMe` RTK Query cache; **JWT on every cold start and after login** |
| **HomeFeed, ContinueWatching, Container, Card** | Memory only | RTK Query cache; overwritten by fetches |
| **Artwork files** | App bundle | placeholder-art assets; Card stores URIs |
| **Catalog source** | In-process mock (`axios-mock-adapter` on the axios instance) | TS/JSON fixtures inside the API module |
| **Page cursors** | Memory | In the RTK Query cache with the page args (ADR-0005 / ADR-0020) |

Persisting `/me` or the catalog cache was considered and **rejected**: cold start already waits on `getMe`, the app is online-only, and a stale `userName` would never be shown first.

---

## 4. Identifiers

| Entity | Key | Exposed? |
|--------|-----|----------|
| **User** | UUID `sub` from the mock JWT | In `/me`. UI shows **`userName` only** |
| **Card, Container** | UUID v4 strings | Yes |
| **ContinueWatching card** | Card `id` (plus implicit session) | No separate id |
| **Pages** | Opaque `nextCursor` (nullable) | Wire only |

No natural keys (`userName` / email are not ids). No auto-increment ints. Same UUID strings in fixtures and the future backend. **Envelope and cursor JSON names are locked in doc 11 §6.1–§6.2** — `{ data, nextCursor }`, with a cursor at each of the two paging levels.

---

## 5. PII / sensitive data, retention, and deletion

Phase 1 has **no real people**. Treat the *shape* as production so it is not logged or leaked.

| Data | Sensitivity | Retention | Deletion |
|------|-------------|-----------|----------|
| Email + password | Confidential | Request lifetime only | Never stored, never logged |
| JWT | Confidential | Until logout, `/me` 401, or expiry | Logout; failed `/me`; uninstall (best-effort — iOS Keychain can survive uninstall) |
| `userName` | Confidential (PII-shaped fixture) | This process only | Logout or process death |
| Continue-watching progress | Confidential in the real product; fixture in Phase 1 | Until the next CW fetch overwrites, or logout | Silent reload replaces the row; logout `resetApiState()` |
| Card, Container, artwork URIs | Internal | In memory / binary | Next fetch or next app release |
| Analytics stubs | Internal | `console.log` only | No PII in stub payloads |

**None of this is regulated** (no payments, health, children’s data, real accounts).

**Privacy/compliance session:** remains **Deferred**. Revisit **before Phase 3 real accounts / real JWT**. Security 1.6 is **Done** (abbreviated) — see `architecture/06-security-threat-model.md`.

### Mock JWT expiry (demo only)

The mock JWT has **`exp` = 7 days from mint**. If it is expired, `/me` fails, the auth slice is cleared, and the user sees login. The **next successful mock login always mints a new token with a new `exp`**. There is **no refresh-token flow**. This logic lives in the **mock adapter** and is **removed when the real API lands**.

---

## 6. Fetch topology and composition

Authenticated calls send the JWT (API-module interceptor).

| # | Call | When | Loading UX |
|---|------|------|------------|
| 1 | `GET /me` (path locked in doc 11 §5) | Cold start with a token; after login | Part of the **shell loading screen** |
| 2 | HomeFeed (`Container[]`, hero + 15) | Same gate as `/me` | Same loader |
| 3 | ContinueWatching (`Container[]`, `variant: "progress"`) | Same gate as `/me`; **again** whenever the storefront screen is shown | Cold start: loader. Later visits: **silent** (stale-while-revalidate; replace that container only; no full-screen loader) |

**Cold start / post-login first paint:** the shell keeps a loading screen until **all three** succeed. Then it paints the composed home. A `/me` 401 clears the session and returns to Welcome. HomeFeed or CW failure is a storefront error state, not the no-internet overlay (doc 15).

**Why wait for all three:** inserting continue watching after HomeFeed has already painted would shift the 15 rows down. `userName` would also flash empty. This is a visual-fidelity demo; a short loader beats a layout jump.

**Silent CW reload:** on storefront appearing again (tab/focus), refetch ContinueWatching only; replace the **`progress` container**. Hero and the other HomeFeed containers stay.

---

## 7. Consistency and schema evolution

| Situation | Consistency |
|-----------|-------------|
| Cold start / first storefront after login | **Strong:** loader until `/me` + HomeFeed + CW complete |
| Continue watching while home is already shown | **Stale-while-revalidate** on the `progress` container only |
| Auth | **Strong:** token present or not; 401 / expiry clears JWT + user + content |

There is **no SQL schema**. Types in the API module *are* the schema (ADR-0016). Additive optional
fields are fine; renames, removals, and type changes are breaking and bump doc 11's Version Log —
the full compatibility rule is doc 11 §4.3. Breaking changes stay free until the backend team
accepts the contract (OQ-34).

**redux-persist `version: 1`** on the auth slice. An incompatible persisted token shape → **purge** the session (safer than a migrate for this POC).

Phase 3 replaces the mock adapter (including mock `exp` reminting). Client storage map stays: persist token, refetch `/me`, memory-only catalog.

---

## Decision Summary

| # | Decision | Choice | Rationale | Forecloses / tradeoff |
|---|----------|--------|-----------|-----------------------|
| 1 | Identity noun | **User** via **`getMe` cache**; UI field is **`userName`** from `/me` | JWT `/me` maps cleanly; login does not invent a profile | Email/id as chrome; stuffing profile into the auth slice; a separate user slice |
| 2 | Artwork | Map of aspect-ratio → URI on **Card** | Fixtures, not a DAM | Artwork as its own entity |
| 3 | Live | **Out** of this model | Later feature | LiveBroadcast / `'live'` variant in Phase 1 data |
| 4 | Catalog split | **HomeFeed** and **ContinueWatching** as two JWT GETs, both **`Container[]`** with **`resources: Card[]`**; client inserts the `progress` container under hero | Same types for UI reuse; CW can silent-reload alone; production APIs stay split | One home payload; CW mixed into the HomeFeed response; distinct CW-only components |
| 5 | Storage | Encrypted persist **auth slice only**; `/me` + catalog in **RTK Query cache** (memory) | Loader waits on `getMe`; online-only; DF3 unchanged | Persisting userName or the API cache; SQLite/MMKV; a server DB |
| 6 | Identifiers | UUID v4 / JWT `sub`; opaque **`nextCursor`** | Stable across a backend swap | Int ids; `userName` as id; offset/`page` |
| 7 | PII | Shape is confidential; nothing regulated | Fake fixtures, internal demo | Privacy session in Phase 1 |
| 8 | Privacy session | **Deferred** until Phase 3 real accounts | No data subjects yet | GDPR/CCPA design now |
| 9 | Mock JWT TTL | **7-day `exp`**, remint on next mock login; no refresh | Exercise expiry → login without a refresh machine | Refresh tokens in Phase 1; immortal mock tokens |
| 10 | First paint | Shell loader until **all three** boot fetches complete | No layout jump when CW inserts under hero | Progressive home as soon as HomeFeed returns |
| 11 | Schema evolution | TS types + persist `version: 1` (purge on break) | No DB to migrate | SQL migrations; clever persist migrate |

## Open Questions

| ID | Question | Owner | Feeds into |
|----|----------|-------|------------|
| ~~OQ-22~~ | ~~JSON names and paths for `/me`, HomeFeed, ContinueWatching, Container, Card, and the page envelope~~ **Resolved (1.11):** five operations and full payload shapes in doc 11 §5, §7. `Card.name` → **`title`** | — | closed |
| ~~OQ-26~~ | ~~Remaining **Card** fields beyond content name, and which are `progress`-only~~ **Resolved (1.11):** §1.2's working set is the Card; §1.3's three fields are optional and `progress`-only (doc 11 §7.1) | — | closed |
| ~~OQ-23~~ | ~~Cards per container horizontal page~~ **Resolved (1.11):** `limit` defaults — 16 on HomeFeed, 10 on Continue Watching and `resources` (doc 11 §6.3) | — | closed |
| ~~OQ-24~~ | ~~Does Phase 1a render full hero chrome or a 3:4 stand-in?~~ **Resolved (1.14):** 1a ships a **3:4 portrait stand-in**; full spotlight chrome is Phase 2. Data composition still includes `variant: "hero"`. | — | closed |
| ~~OQ-25~~ | ~~Exact mock `/me` payload beyond `id` + `userName` (claims vs body)~~ **Resolved (1.6a):** mock JWT claims = `sub` + `exp` + `iat`; `/me` = `{ id, userName }`. JSON names → OQ-22 | — | closed |

**OQ-02** (card schema), **OQ-19** (cursor vs offset), and HomeFeed first-page size from **OQ-20** (hero + 15; CW separate) were closed here in earlier revisions; **OQ-17**, **OQ-22**, **OQ-23**, and **OQ-26** are closed by 1.11. **OQ-24** is closed by 1.14. Carried forward: **OQ-30** (server-localized `Container.name`) and **OQ-34** (backend accepts the contract — doc 11 §14). Identity living doc is `architecture/16-identity-auth.md`; the wire contract is `architecture/11-interface-contracts.md`.

## Version Log

| Version | Date | STEP | Change |
|---------|------|------|--------|
| v0.1.0 | 2026-08-16 | STEP-1.4 | Initial draft from the data-model session |
| v0.2.0 | 2026-08-17 | STEP-1.5 | Shared **Container** / **Card**; both feeds are `Container[]` with `resources`; variants `hero` and `progress`. Title/Carousel/`items` renamed. ADR-0007. |
| v0.2.1 | 2026-08-17 | STEP-1.6 | Privacy session still Deferred; security posture now in doc 06 (abbreviated, Done). |
| v0.2.2 | 2026-08-17 | STEP-1.6a | Closed OQ-25 (`/me` = `{ id, userName }`; JWT claims `sub`/`exp`/`iat`). Doc 16. |
| v0.2.3 | 2026-08-17 | STEP-1.7 | §1.4 corrected: `Container.name` is a localized display string from the wire, not client-side i18n (doc 07 §9). Opened OQ-30. |
| v0.2.4 | 2026-08-17 | STEP-1.11 | **`Card.name` → `Card.title`.** Cursor placement resolved as *both* levels; unknown-`variant` rule added; `expiresAt` is ISO on the wire; paths and envelope point at doc 11. Closed OQ-22, OQ-23, OQ-26. |
| v0.2.5 | 2026-08-17 | STEP-1.14 | User and content slices replaced by RTK Query cache (`getMe` + feeds). Catalog mocks are `axios-mock-adapter` on the axios instance (ADR-0020). |
| v0.2.6 | 2026-08-17 | STEP-1.14 | Closed OQ-24 (3:4 hero stand-in). `visibleCount` is client-side, not a Container field. |
