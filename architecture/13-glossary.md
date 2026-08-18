# Doc 13 — Glossary

**Version:** v0.2.3
**Status:** Draft
**Last updated:** 2026-08-18 (STEP-2.2)
**Audience:** Mobile developers, backend team, QA, future agents

> Precise meaning of quasar-disney-mobile's domain terms, entities, acronyms, and naming
> rules — harvested from the architecture docs so later STEPs use the same words.

This is a **living** doc. Add a row when a later STEP introduces a durable term; bump the
Version Log. Entity names align with `architecture/04-data-model.md`. Wire names align with
`architecture/11-interface-contracts.md`.

The packaging and dependency rule this glossary names — **feature-based Clean Architecture**
inside a **modular monolith** — is already how docs 03–04 and 11 are built. This session did
not redesign folders; it locked the vocabulary so a later STEP does not drift into a
layer-first tree or a feature importing another feature.

## Table of Contents

1. [Term table](#1-term-table)
2. [Clean Architecture mapping](#2-clean-architecture-mapping)
3. [Acronyms](#3-acronyms)
4. [Naming conventions](#4-naming-conventions)

---

## 1. Term table

Alphabetical. The Notes column is the disambiguation: what the term is *not*.

| Term | Definition | Notes / not to be confused with |
|------|------------|----------------------------------|
| **Account** | *Do not use.* | Say **User** + **Session**. |
| **`@env`** | Build-time config module. Keys live in a gitignored `.env`; `.env.example` is the informal contract. | Not a runtime remote-config service. |
| **API module** | The only I/O boundary: RTK Query `baseApi`, axios client + interceptors, `axios-mock-adapter`, wire types. Screens never import `axios` or `fetch`. | Not a backend. Not a mock HTTP server. |
| **`baseApi`** | `createApi` instance in `src/api/baseApi.ts` with empty endpoints. Features `injectEndpoints`. Shell registers its reducer and middleware. | Not a backend. Not handwritten async middleware. |
| **Auth feature** | Welcome → email → password → session. Owns the auth slice; injects `login` / `getMe`. Imports `shared/` and `src/api/types/` only. | Must not import Storefront. Does not store Credentials. |
| **Auth slice** | RTK slice holding `accessToken` + `expiresAt`. The only persisted slice. | Not the RTK Query cache. |
| **`ApiError`** | Normalized error both transports produce: `{ code, status, message }`. Features never see a raw axios error. | `message` is developer-facing and never rendered. |
| **App shell** | Composition root. Boots RN: navigation, store (`baseApi` + auth slice), Emotion `ThemeProvider`, persist rehydrate, cold-start loader, NetInfo overlay, root error boundary. Only module allowed to import features. | Not a feature. Not “the app” as a whole. |
| **Artwork** | Aspect-ratio → URI map on **Card**. Bundled placeholder files, not an entity. | Keys: `'2:3'`, `'16:9'`, `'3:4'`. |
| **Atom / molecule / organism** | Atomic-design UI layers. Lives in `shared/ui/` iff two or more of {auth, storefront, shell} render it; otherwise with its feature. | Not architecture “components” (the five modules). |
| **Auth feature** | Welcome → email → password → session. Owns the auth slice; injects `login` / `getMe`. Imports `shared/` and `src/api/types/` only. | Must not import Storefront. Does not store Credentials. |
| **Auth slice** | RTK slice holding `accessToken` + `expiresAt`. The only persisted slice. | Not the RTK Query cache. |
| **Card** | Catalog item a tile represents. Shared across container variants. Was called **Title**. | Not the UI **tile**. Not a Container. `Card.title` is the content-name field. |
| **Carousel** | Config-driven UI that renders a Container. Variants are config, not extra components. | Not a data entity. |
| **Clean Architecture** | Dependency rule: inner (feature hooks, slices, models) does not depend on outer (axios, mocks, RN host, persist, NetInfo). Screens never fetch. Shell composes; `baseApi` is the I/O adapter. | Not a `domain/usecases/data` tree inside every feature. Wire types still live in `src/api/types/` because the contract is the schema. |
| **Cold-start loader** | Shell-owned full-screen wait until `/me` + HomeFeed + Continue Watching all succeed, then first paint of the composed home. | Not a storefront spinner. Not the no-internet overlay. |
| **ComingSoon** | Placeholder screen for inert tabs (Buscar, Descargas, Perfil). Tappable so a dead tap is not mistaken for a bug. | Not a deferred feature stub with real data. |
| **Composed home** | Client-only merge owned by Storefront: HomeFeed hero + CW `progress` container + remaining HomeFeed containers. Not an API resource. | Not HomeFeed. |
| **Composition root** | The shell’s registration role: screens, reducers, middleware. | Not a sixth module. |
| **Connectivity gate** | Shell-owned NetInfo overlay matching the no-internet reference. Sits on top of the current navigator; does not unmount it. | Not a feature fetch error. **Online = interface up** (no reachability probe). |
| **Container** | A home row: `name`, `variant`, `resources: Card[]`, optional `nextCursor`. Same type on both feeds. | Not the carousel component. Unrecognized `variant` drops the row with `console.warn`. |
| **Content cache** | In-memory RTK Query copy of catalog + pagination. Overwritten by fetches; never persisted. | Owned by Storefront endpoints. Replaces the former content slice. |
| **Continue Watching (CW)** | Paginated `Container[]` from a second JWT GET. Typically one container, `variant: "progress"`. Same types as HomeFeed; never mixed into the HomeFeed response. | Progress fields are optional on Card, populated only in a `progress` container. |
| **Contract of record** | Until the app repo exists: `architecture/11-interface-contracts.md`. After scaffold: the TypeScript wire types; this doc stays the consumer-facing narrative until OpenAPI is triggered. | Not OpenAPI today (ADR-0016). |
| **Credentials** | Login request DTO `{ email, password }`. Request-lifetime only; never stored, never logged. | Not an entity. Not part of User. |
| **DF (don’t-foreclose)** | Phase-1 constraint that must be respected even though the demo doesn’t fully exercise it yet (doc 02 §6). | Not a RISK. |
| **Dinsey-** | Fictional placeholder brand for the internal POC. Assets in `architecture/assets/brand/`. | Not Disney. Rename before any public release (RISK-0005). |
| **Emotion** | CSS-in-JS stack: `@emotion/native` (`styled`) + `@emotion/react` (`ThemeProvider`, `useTheme`). Token palette lives in `shared/theme/`. | Not `styled-components`. |
| **Environment** | A named **build configuration** plus the config values the app is built with — `development` or `release`. Not a hosting destination. | No staging in Phase 1. CI is a runner of `release`, not a third env. |
| **Envelope** | Page wrapper `{ data, nextCursor }`. Vertical cursor pages containers; each Container’s cursor pages its `resources`. | Not a bare JSON array. |
| **Feature** | Extractable product surface (`auth`, `storefront`). Colocates UI, hooks, slices, feature types. May not import another feature. | Prefer the feature name in prose over “module.” |
| **Feature-based** | Packaging rule: organize by product surface under `src/features/*`, not a repo-root layer tree (`screens/`, `redux/`, `api/` as siblings of everything). Implements Clean Architecture in this codebase. | Compatible with the five-module modular monolith. |
| **Fixtures** | Typed demo catalog/auth data the mock adapter serves. Same fixtures in `development` and `release`. | Tests assert against **factories**, not fixtures (one fixture-invariant test excepted). |
| **Hero** | Container `variant: "hero"`. First item on HomeFeed page 1. Phase 1a renders a **3:4 portrait stand-in**; full spotlight chrome is Phase 2 (OQ-24 closed). | Not a separate entity. |
| **HomeFeed** | Paginated `Container[]` from `GET /home-feed`. First page: one `hero` + 15 other containers. | Not the composed home the user sees. |
| **IdP** | Identity provider. None in Phase 1. Phase 3 **buys** a managed IdP **behind our API**, not as an RN SDK. | Not Auth0/Firebase in the app. |
| **Inert tab** | Tab that is tappable and routes to **ComingSoon**. | Not a disabled/unresponsive tab. |
| **`INVALID_CREDENTIALS`** | Login failure code. Credentials screen shows the inline error. Does **not** clear a session. | Not `UNAUTHORIZED`. One code for both “no such email” and “wrong password.” |
| **`items`** | *Do not use* for cards in a row. | The field is **`resources`**. |
| **JWT / access token** | Opaque session token in the auth slice. Mock claims: `sub`, `exp`, `iat`. 7-day mock TTL; reminted on next mock login. No refresh token in Phase 1. | Client never decodes the JWT to learn expiry — `expiresAt` is on the wire as ISO 8601 UTC. |
| **`limit`** | Request page-size hint. Defaults: **16** HomeFeed, **10** Continue Watching and `resources`. Server may cap it. | Not a guarantee of page length. |
| **Middleware** | RTK Query's `baseApi.middleware`. Features reach I/O through generated hooks, not handwritten async middleware. | Not `createAsyncThunk`. Not HTTP middleware. Not axios interceptors (those live on the axios instance). |
| **Mock adapter** | `axios-mock-adapter` on the shared axios instance. Promises, ~400–600 ms latency, injectable failure. A **production artifact**, not a test double. Replaced in Phase 3 by dropping the adapter. | Not Jest mocks. Not fixtures (those are the data it serves). Not a separate mock client that bypasses interceptors. |
| **Modular monolith** | One RN app, five in-process modules, hard import rules. The runtime shape that carries feature-based Clean Architecture. | Not a backend monolith. We build no server. |
| **`nextCursor`** | Opaque pagination cursor (nullable). Wire only; not an entity. | Not offset/`page`. Two levels: envelope (vertical) and Container (horizontal). |
| **Phase 1a** | Demo-gated cut for **2026-08-18** stakeholder sign-off. | Phase 1 is not complete on that date — **1b** is the remainder. |
| **Phase 1b** | Rest of Phase 1, no external date: remaining carousel variants, UI tests, formal QA checklist, polish. | Not Phase 2. |
| **Phase 2** | Complete storefront chrome (hero/filter rail/details) + Bitrise. No backend required. | |
| **Phase 3** | Backend integration. **We never build a backend.** Mock adapter → real HTTP; real JWT. | Externally gated on the backend team. |
| **Phase 4+** | Playback, profiles, downloads, parental controls, settings, payments. | None of these change a Phase-1 decision. |
| **POC** | What Phase 1 is: a functional stakeholder demo with visual fidelity. No end-user value claimed. | **Not an MVP.** |
| **Profile** | Deferred household-profile feature. | Do not use as a synonym for User. |
| **Release build** | The **sign-off artifact**: embedded JS bundle, no Metro, USB-installed. Maps to the `release` environment. | Debug/Metro is the development loop, not what stakeholders see. |
| **`resources`** | `Card[]` inside a Container (one horizontal page). | Not `items`. Horizontal paging: `GET /containers/{id}/resources`. |
| **Root error boundary** | Shell-owned React boundary inside the theme provider. A render exception degrades to a recoverable screen, not a white screen. | Not a storefront error state. Not the no-internet overlay. The only observability code Phase 1 adds. |
| **Session** | The logged-in token: `accessToken` + `expiresAt`. Auth slice. The only persisted entity. | Not the User. No refresh token in Phase 1. |
| **Shared kernel** | Theme (both surface modes), atomic UI, i18n tables, analytics stub. Folders: `theme/`, `ui/`, `i18n/`, `analytics/`. **No types package.** | Wire types live in the API module, not here. |
| **Silent CW reload** | When storefront is shown again, refetch Continue Watching only and replace the `progress` container. Stale-while-revalidate; no full-screen loader. | Not a cold-start. Hero and other HomeFeed rows stay. |
| **Slice** | An RTK state partition. Phase 1 persisted slice: **auth**. Catalog and `/me` live in the **RTK Query cache** (`baseApi`). | Not a feature. |
| **Storefront feature** | Home/browse and the config-driven carousel. Injects feed/resources endpoints; pagination wrappers, composed-home merge, silent CW reload. | Not HomeFeed. Must not import Auth. |
| **Surface mode** | Theme axis named by **role**: `app` (dark) and `auth` (light). Same token keys in both. | **Not** `light`/`dark` and **not** `useColorScheme()`. Auth stays light because the reference sheet is light. |
| **Test factory** | `makeCard` / `makeContainer` / `makePage` in `src/api/mocks/`. Tests-only; must not be reachable from the app entry. | Not demo fixtures. |
| **Theme** | Brand token set. Default brand is `dinsey`; a second test theme (`ember`) exists so criterion A1 (re-skin without touching components) is verifiable. | Theme × surface mode are two axes. |
| **Tile** | UI molecule that renders a Card. | Not a data entity. |
| **Title** | Retired entity name for Card. | `Card.title` is now just the content-name field. |
| **Token (design)** | A named theme value (color, type, space, motion). No hardcoded colors/type/spacing outside the theme. | Not a JWT. |
| **Token (auth)** | The JWT / access token. | Say **JWT** or **access token** when the auth token is meant. |
| **Trademark substitution** | Reproduce layout and interaction; never Disney marks or real key art. Placeholders at identical aspect ratios. | Binding at every phase (DF10). |
| **`UNAUTHORIZED`** | 401 on operations 2–5 (`/me`, feeds, resources). Middleware clears the session → Welcome. | Not `INVALID_CREDENTIALS`. |
| **User** | Identity the UI can show: JWT `sub` as `id` plus `userName` from `GET /me`. Memory-only (`getMe` cache). | Not an account, not a **Profile**, not the login email. |
| **User cache** | RTK Query cache for `getMe`: `{ id, userName }`. Refilled on every cold start and after login. | Not persisted. Not inside the auth slice. |
| **Variant** | Container enum: `'hero'`, `'progress'`, `'standardPortrait'`, `'standardLandscape'`. `'progress'` is not a HomeFeed member. `'live'` is out until that feature lands. | Config on Container, not a TypeScript type per layout. |
| **Visual fidelity** | Explicit Phase-1 goal: layout, spacing, structure, interaction matching the reference — with substituted brand and art. | Not pixel-perfect Disney IP. |
| **Wire types** | Request/response TypeScript shapes in **`src/api/types/`**. Authoring source of the contract once the repo exists. Features may import **these types only** — never `src/api/client` or `src/api/mocks`. | Not in `shared/`. Feature-local types stay in the feature; component props stay next to the component. |

---

## 2. Clean Architecture mapping

How the five in-app modules map onto Clean Architecture roles. This is naming, not a new
folder tree.

| Clean Architecture role | Where it lives here |
|-------------------------|---------------------|
| Frameworks & drivers | App shell (RN, navigation, persist, NetInfo, error boundary) |
| Interface adapters | API module (`baseApi`, axios + interceptors + mock adapter + **wire types**), feature `injectEndpoints`, auth slice |
| Use cases | Feature hooks (`loadMore`, login submit, composed-home merge, silent CW reload) wrapping RTK Query hooks |
| Entities | `User`, `Session`, `Card`, `Container` (TypeScript models; wire forms owned by `src/api/types/`) |

**Types-only import rule** (the dependency rule that matters):

| Who | May import | Must not import |
|-----|------------|-----------------|
| Features (hooks, screens) | `src/api/types`; own `src/features/<name>/api.ts` hooks | `src/api/client`, `axios` |
| API client / mocks / `baseApi` | `src/api/types` | features |
| Shared kernel | nothing from `src/api/` | — |

Features depending on **schema types** is not I/O. Features depending on the **adapter** is.

---

## 3. Acronyms

| Acronym | Means |
|---------|--------|
| **ADR** | Architecture decision record (`adr/ADR-NNNN-*.md`) |
| **ATS** | App Transport Security — Phase 3 HTTPS posture |
| **CI** | JS test gate in Phase 1a; Bitrise native builds in Phase 2 |
| **CW** | Continue Watching |
| **DTO** | Request/response object, never stored |
| **JWT** | Access token (no refresh token in Phase 1) |
| **OQ** | Open question, tracked in architecture docs |
| **PII** | Treated by *shape* in Phase 1; no real people |
| **POC** | Proof of concept — Phase 1 milestone type, not an MVP |
| **RN** | React Native (bare, no Expo) |
| **RTK** | Redux Toolkit |
| **TTL** | Mock JWT lifetime: 7 days from mint |

---

## 4. Naming conventions

Consistent with `METHOD.md` §8 for process artifacts; the rest is this project's code and wire
shape.

**Folders.** `src/app/` (shell), `src/features/<name>/` (feature-based, including `api.ts` injectEndpoints),
`src/shared/{theme,ui,i18n,analytics}/`, `src/api/{baseApi.ts,types,mocks,client}/`. No `modules/` prefix.
No repo-root layer tree. No `shared/types/` package.

**Entities.** PascalCase (`User`, `Card`, `Container`). IDs are UUID v4 strings (User id = JWT
`sub`). No natural keys, no auto-increment ints.

**JSON / wire.** camelCase. `Container.name` vs `Card.title`. `resources` not `items`. Envelope
`{ data, nextCursor }`. ISO 8601 UTC for `expiresAt` on the wire.

**Code.** Features never import features; only the shell composes. I/O only through RTK Query hooks
on `baseApi`. Features may import `src/api/types` and their own `api.ts`; never `src/api/client`.

**Do not use.** Account (say User + Session); Title as an entity; `items` for cards; Expo;
fetch-from-screens; `styled-components`; `createAsyncThunk` for I/O; a global types module in `shared/`.

**Process.** `STEP-N` unpadded in prose; `step-NNNN-short-name` for branches. ADRs
`ADR-NNNN-kebab-title`.

**UI copy.** No loose hardcoded strings (DF8). `Container.name` is feed-supplied data, not
client i18n. Surface modes are `app` / `auth`, never `light` / `dark`.

---

## Decision Summary

| # | Decision | Choice | Rationale | Forecloses / tradeoff |
|---|----------|--------|-----------|-----------------------|
| 1 | Glossary posture | Living term table harvested from all `architecture/` docs, including conditionals 15 and 16 | Prevents agent/human drift on overloaded words (User vs Profile, Card vs tile, HomeFeed vs composed home) | A frozen glossary that later STEPs cannot extend |
| 2 | Architecture names | **Feature-based Clean Architecture** inside a **modular monolith** — naming the shape docs 03 already specified | User asked it be explicit from the start; folders do not change | A textbook `domain/usecases/data` tree inside every feature |
| 3 | Wire types location | Stay in **`src/api/types/`**, not `shared/` | API module owns the contract; doc 03 already rejected a global types module; extraction takes types with the adapter | Stricter “inner ring” CA with duplicated domain vs wire models |
| 4 | Types import rule | Features may import **schema types only**; never `client/` or `mocks/` | Keeps the Clean Architecture dependency rule without a mapper layer | Features importing axios or the mock adapter |
| 5 | Entity nouns | Align with doc 04: User, Session, Card, Container, HomeFeed, Continue Watching. Composed home is client-only | Data model is the source of truth | Reviving Title / Account / `items` |
| 6 | Surface-mode names | Role names `app` / `auth`, never `light` / `dark` | Matches ADR-0011; `useColorScheme()` would break the light auth sheet | OS-driven dark mode in Phase 1 |
| 7 | Token disambiguation | “Token” in UI docs = design token; auth token is **JWT** / **access token** | The word is overloaded in this codebase | Casual “token” in mixed auth+theme prose |

---

## Open Questions

| ID | Question | Owner | Feeds into |
|----|----------|-------|------------|
| — | None opened by this session. The doc is living: add terms when later STEPs introduce them. | — | — |

**Handoff for 1.14:** echo Decision 2 in doc 03 §2 — **done** (doc 03 v0.4.0). Also Emotion + RTK Query `baseApi` (ADR-0019, ADR-0020).

Carried naming leftovers (not coined here): **OQ-18** is closed — application repo is
`quasar-disney-mobile-app` at `Code/quasar-disney-mobile-app/` (registered STEP-2.2).

---

## Version Log

| Version | Date | STEP | Change |
|---------|------|------|--------|
| v0.1.0 | 2026-08-17 | STEP-1.13 | Initial glossary from all architecture docs through 1.12. Feature-based Clean Architecture named; wire types kept in `src/api/types/` with a types-only import rule. |
| v0.2.0 | 2026-08-17 | STEP-1.14 | Emotion + RTK Query `baseApi` terms. User/content slices → cache. 1.14 handoff for doc 03 §2 closed. |
| v0.2.1 | 2026-08-17 | STEP-1.14 | Hero: 3:4 stand-in in 1a (OQ-24 closed). |
| v0.2.2 | 2026-08-17 | planning session | OQ-18 closed: application repo is `quasar-disney-mobile-app`. |
| v0.2.3 | 2026-08-18 | STEP-2.2 | App repo exists at `Code/quasar-disney-mobile-app/`. |
