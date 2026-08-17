# Doc 04 — Data Model, Ownership & Retention

**Version:** v0.2.0
**Status:** Draft
**Last updated:** 2026-08-17 (STEP-1.5)
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

### 1.2 Card (closes OQ-02; extra fields still open as OQ-26)

The wire **must** include a content **name**. Other production fields are **TBD** (1.11). Phase 1 mocks still fill the UI working set below so hero / portrait / `progress` chrome can ship.

| Field | Type | Notes |
|-------|------|--------|
| `id` | UUID string | Public; tiles, later details, CW merge |
| `name` | string | Content name. Shown on tap-alert; CW and landscape rows. JSON key → 1.11 (may be `title`) |
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

Whether these stay on Card, are `progress`-only, or get different JSON names is **OQ-26**. The Continue Watching **request sends the JWT**. The mock (and later the backend) keys progress off that session. There is no WatchProgress id.

### 1.4 Container (HomeFeed and Continue Watching)

Same type on **both** authenticated feed GETs (ADR-0007).

| Field | Type | Notes |
|-------|------|--------|
| `id` | UUID string | Row identity for paging cards |
| `name` | string | Row header; i18n in the client. JSON key → 1.11 |
| `variant` | enum | `'hero' \| 'progress' \| 'standardPortrait' \| 'standardLandscape'`. **`'progress'` is not a HomeFeed response member** — it comes from the CW GET. `'live'` is out until that feature lands |
| `resources` | Card[] | One page of cards (not `items`). Envelope `nextCursor` pages the list of containers (vertical) or a container’s resources (horizontal) — 1.11 names the JSON |
| `nextCursor` | string \| null | Opaque; Phase 1 mocks may send `null`. Exact placement (container vs page envelope) → 1.11 |

HomeFeed first page: **`Container[]`** with one `variant: "hero"` plus **15** other containers. Further vertical pages, if any, are more containers only (no second hero). Phase 1 mocks may set HomeFeed `nextCursor` to `null`.

Continue Watching: **`Container[]`** (typically length 1), each with `variant: "progress"` and `resources: Card[]`.

### 1.5 User and Session

| Field | Type | Notes |
|-------|------|--------|
| User.`id` | UUID | JWT `sub`; not shown in UI |
| User.`userName` | string | The only profile field the UI shows. Comes from `/me`, not from login |
| Session.`accessToken` | string | Mock JWT |
| Session.`expiresAt` | number | Unix `exp`. Mock TTL **7 days** from each successful login |

Login body: `{ email, password }` — request DTO, never stored.

---

## 2. Ownership

| Entity | Authoritative owner | Who else reads it | Notes |
|--------|---------------------|-------------------|--------|
| **Session** | Auth feature (auth slice) | Shell (nav, theme, boot gate) | Persisted |
| **User** | Auth feature (**user slice**) | Shell / chrome if needed; **not** Storefront | Memory only. Auth ↛ Storefront still holds |
| **Card, Container, HomeFeed** | API module (wire + fixtures) | Storefront content slice (in-memory copy) | Storefront never authors catalog |
| **ContinueWatching** | API module (JWT-authenticated fixtures) | Storefront content slice | Same Container/Card types; `progress` is on CW cards, not in Auth |
| **Composed home list** | Storefront feature | — | Inserts the `progress` container under hero (ADR-0006) |
| **Cold-start loading** | App shell | — | Blocks until `/me` + HomeFeed + CW complete |
| **Silent CW reload** | Storefront feature | — | When the storefront screen is shown again |

---

## 3. Storage

No relational database, no SQLite/MMKV, no mock HTTP server acting as a DB.

| Data | Where | Engine |
|------|--------|--------|
| **Session** (JWT + `exp`) | Device, encrypted key-value | redux-persist whitelist = **auth slice only** + `react-native-encrypted-storage` |
| **User** | Memory only | user slice; **`/me` with the JWT on every cold start and after login** |
| **HomeFeed, ContinueWatching, Container, Card** | Memory only | content slice; overwritten by fetches |
| **Artwork files** | App bundle | placeholder-art assets; Card stores URIs |
| **Catalog source** | In-process mock adapter | TS/JSON fixtures inside the API module |
| **Page cursors** | Memory | With the content slice (ADR-0005) |

Persisting the user slice was considered and **rejected**: cold start already waits on `/me`, the app is online-only, and a stale `userName` would never be shown first.

---

## 4. Identifiers

| Entity | Key | Exposed? |
|--------|-----|----------|
| **User** | UUID `sub` from the mock JWT | In `/me`. UI shows **`userName` only** |
| **Card, Container** | UUID v4 strings | Yes |
| **ContinueWatching card** | Card `id` (plus implicit session) | No separate id |
| **Pages** | Opaque `nextCursor` (nullable) | Wire only |

No natural keys (`userName` / email are not ids). No auto-increment ints. Same UUID strings in fixtures and the future backend. Envelope field names for the cursor land in session 1.11 (OQ-19 closed on *kind*; 1.11 names the JSON).

---

## 5. PII / sensitive data, retention, and deletion

Phase 1 has **no real people**. Treat the *shape* as production so it is not logged or leaked.

| Data | Sensitivity | Retention | Deletion |
|------|-------------|-----------|----------|
| Email + password | Confidential | Request lifetime only | Never stored, never logged |
| JWT | Confidential | Until logout, `/me` 401, or expiry | Logout; failed `/me`; uninstall (best-effort — iOS Keychain can survive uninstall) |
| `userName` | Confidential (PII-shaped fixture) | This process only | Logout or process death |
| Continue-watching progress | Confidential in the real product; fixture in Phase 1 | Until the next CW fetch overwrites, or logout | Silent reload replaces the row; logout clears the content slice |
| Card, Container, artwork URIs | Internal | In memory / binary | Next fetch or next app release |
| Analytics stubs | Internal | `console.log` only | No PII in stub payloads |

**None of this is regulated** (no payments, health, children’s data, real accounts).

**Privacy/compliance session:** remains **Deferred**. Revisit **before Phase 3 real accounts / real JWT**. Security 1.6 still runs abbreviated.

### Mock JWT expiry (demo only)

The mock JWT has **`exp` = 7 days from mint**. If it is expired, `/me` fails, the auth slice is cleared, and the user sees login. The **next successful mock login always mints a new token with a new `exp`**. There is **no refresh-token flow**. This logic lives in the **mock adapter** and is **removed when the real API lands**.

---

## 6. Fetch topology and composition

Authenticated calls send the JWT (API-module interceptor).

| # | Call | When | Loading UX |
|---|------|------|------------|
| 1 | `GET /me` (name illustrative; 1.11 locks paths) | Cold start with a token; after login | Part of the **shell loading screen** |
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

There is **no SQL schema**. Types in the API module *are* the schema. Additive optional fields are fine; renames need a contract version (session 1.11).

**redux-persist `version: 1`** on the auth slice. An incompatible persisted token shape → **purge** the session (safer than a migrate for this POC).

Phase 3 replaces the mock adapter (including mock `exp` reminting). Client storage map stays: persist token, refetch `/me`, memory-only catalog.

---

## Decision Summary

| # | Decision | Choice | Rationale | Forecloses / tradeoff |
|---|----------|--------|-----------|-----------------------|
| 1 | Identity noun | **User** + separate **user slice**; UI field is **`userName`** from `/me` | JWT `/me` maps cleanly; login does not invent a profile | Email/id as chrome; stuffing profile into the auth slice |
| 2 | Artwork | Map of aspect-ratio → URI on **Card** | Fixtures, not a DAM | Artwork as its own entity |
| 3 | Live | **Out** of this model | Later feature | LiveBroadcast / `'live'` variant in Phase 1 data |
| 4 | Catalog split | **HomeFeed** and **ContinueWatching** as two JWT GETs, both **`Container[]`** with **`resources: Card[]`**; client inserts the `progress` container under hero | Same types for UI reuse; CW can silent-reload alone; production APIs stay split | One home payload; CW mixed into the HomeFeed response; distinct CW-only components |
| 5 | Storage | Encrypted persist **auth slice only**; user + catalog **memory** | Loader waits on `/me`; online-only; DF3 unchanged | Persisting userName; SQLite/MMKV; a server DB |
| 6 | Identifiers | UUID v4 / JWT `sub`; opaque **`nextCursor`** | Stable across a backend swap | Int ids; `userName` as id; offset/`page` |
| 7 | PII | Shape is confidential; nothing regulated | Fake fixtures, internal demo | Privacy session in Phase 1 |
| 8 | Privacy session | **Deferred** until Phase 3 real accounts | No data subjects yet | GDPR/CCPA design now |
| 9 | Mock JWT TTL | **7-day `exp`**, remint on next mock login; no refresh | Exercise expiry → login without a refresh machine | Refresh tokens in Phase 1; immortal mock tokens |
| 10 | First paint | Shell loader until **all three** boot fetches complete | No layout jump when CW inserts under hero | Progressive home as soon as HomeFeed returns |
| 11 | Schema evolution | TS types + persist `version: 1` (purge on break) | No DB to migrate | SQL migrations; clever persist migrate |

## Open Questions

| ID | Question | Owner | Feeds into |
|----|----------|-------|------------|
| OQ-22 | JSON names and paths for `/me`, HomeFeed, ContinueWatching (the `Container[]` attribute), Container (`name`, `resources`), Card, and the page envelope (`nextCursor` vs `next`) | Mobile | 1.11 Interface Contracts |
| OQ-26 | Remaining **Card** fields beyond content name, and which are `progress`-only | Mobile | 1.11; 1.7 |
| OQ-23 | Cards per container horizontal page (first-page size for `resources` inside a row) | Mobile | 1.11; storefront STEP |
| OQ-24 | Does Phase 1a render full hero chrome (peeking neighbors, title art, CTA) or a 3:4 stand-in? Data composition already includes hero | Mobile / 1.7 | 1.7 UI / Design System; planning session |
| OQ-25 | Exact mock `/me` payload beyond `id` + `userName` (claims vs body) | Mobile | 1.6a Identity & Auth; 1.11 |

Carried forward: OQ-10 (backend team accepts the contract → 1.11), OQ-17 (mock strategy → 1.11). **OQ-02** (card schema), **OQ-19** (cursor vs offset), and HomeFeed first-page size from **OQ-20** (hero + 15; CW separate) are **closed** here. Horizontal **card** page size remains as OQ-23. Extra Card fields → OQ-26.

## Version Log

| Version | Date | STEP | Change |
|---------|------|------|--------|
| v0.1.0 | 2026-08-16 | STEP-1.4 | Initial draft from the data-model session |
| v0.2.0 | 2026-08-17 | STEP-1.5 | Shared **Container** / **Card**; both feeds are `Container[]` with `resources`; variants `hero` and `progress`. Title/Carousel/`items` renamed. ADR-0007. |
