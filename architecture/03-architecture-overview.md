# Doc 03 — Architecture Overview & Component Boundaries

**Version:** v0.4.7
**Status:** Draft
**Last updated:** 2026-08-19 (login screen styling rule)
**Audience:** Mobile developers, backend team, QA

> How quasar-disney-mobile is cut into components, how those pieces talk, and which boundary is the only one that needs a formal contract.

## Table of Contents

1. [Client surfaces](#1-client-surfaces)
2. [Architecture style](#2-architecture-style)
3. [Component diagram](#3-component-diagram)
4. [Component table](#4-component-table)
5. [Boundaries & contract candidates](#5-boundaries--contract-candidates)
6. [Key flows](#6-key-flows)
7. [Build vs. buy](#7-build-vs-buy)
8. [Import rules & source layout](#8-import-rules--source-layout)

---

## 1. Client surfaces

**One surface:** a **React Native mobile app on iOS and Android**. No web, desktop, CLI, or public API-as-a-product.

| Gate | Call |
|------|------|
| **UI / Design System (1.7)** | **Include** — styled screens are the demo |
| **Native-app session (1.3a)** | **Include** — already slotted; covers offline, permissions, pinning, distribution |

A companion web/admin surface or treating the API as a standalone product can land later without rewriting the mobile boundary.

## 2. Architecture style

**Feature-based Clean Architecture** inside a **modular monolith** — one React Native app, five in-process modules, hard import rules. Organize by product surface (`src/features/*`), not a repo-root layer tree. The **app shell is the composition root**: it is the only module allowed to import features in order to register screens, reducers, and RTK Query middleware.

Inner layers (feature hooks, slices, models) do not depend on outer ones (axios, mocks, RN host, persist, NetInfo). Screens never fetch. Wire types live in `src/api/types/`; features may import those types only, plus their own `injectEndpoints` hooks. This is naming for the shape already specified here — not a `domain/usecases/data` tree inside every feature.

This is the simplest shape that still makes auth and storefront extractable later (doc 02 DF5). It **forecloses** splitting into services, a separate mock HTTP server process, or extracting npm packages in Phase 1.

Internal partition of Shared is folders, not packages: `shared/theme/`, `shared/components/`, `shared/i18n/`, `shared/analytics/`. There is **no types module** — wire types live in the API module, feature types live with the owning feature, component props live next to the component.

Code lives in a **single application repo**, **`quasar-qc-plus-mobile-app`** at `Code/quasar-qc-plus-mobile-app/` (OQ-18 closed; registered STEP-2.2; renamed STEP-6.1). No backend repo is created by this project.

## 3. Component diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         App shell                               │
│   RN host · navigation · store · persist rehydrate · NetInfo    │
│                                                                 │
│         ┌──────────────┐         ┌──────────────────┐           │
│         │ Auth feature │         │ Storefront       │           │
│         │              │         │ feature          │           │
│         └──────┬───────┘         └────────┬─────────┘           │
│                │                          │                     │
│                │     (RTK Query hooks)    │                     │
│                └────────────┬─────────────┘                     │
│                             ▼                                   │
│                    ┌─────────────────┐                          │
│                    │   API module    │                          │
│                    │ baseApi + axios │                          │
│                    └─────────────────┘                          │
│                             ▲                                   │
│         ┌───────────────────┴───────────────────┐               │
│         │           Shared kernel               │               │
│         │  theme / ui / i18n / analytics        │               │
│         └───────────────────────────────────────┘               │
│                                                                 │
│   Auth ↛ Storefront. Shell reads auth selectors to switch nav.  │
└─────────────────────────────────────────────────────────────────┘
```

A future HTTP backend is **not a component of this system**. It is a Phase-3 swap behind the API module.

## 4. Component table

| Name | Responsibility | Tech |
|------|----------------|------|
| **App shell** | Boots the RN host on iOS/Android: navigation, store registration (`baseApi.reducer` + `baseApi.middleware` + auth slice), Emotion `ThemeProvider`, persist rehydrate, **cold-start loading** until `getMe` + `getHomeFeed` + `getContinueWatching` complete, **NetInfo connectivity overlay**, **root error boundary** (inside the theme provider; doc 10 §5.2) | Bare RN, TypeScript, React Navigation, Redux Toolkit + RTK Query, `@react-native-community/netinfo` |
| **Auth feature** | Welcome → email → password → session. Owns the **auth slice** (JWT) and injects `login` / `getMe` onto `baseApi`. Imports `shared/` and `src/api/types/` only | RTK slice, RTK Query endpoints, custom hooks, Emotion styled |
| **Storefront feature** | Home/browse and config-driven carousels. Injects feed/resources endpoints; **pagination wrappers**, **client merge** of hero + `progress` (CW) + other containers, **silent CW reload**. Catalog lives in the RTK Query cache (not persisted). Imports `shared/` and `src/api/types/` only | RTK Query endpoints, custom hooks, Emotion styled |
| **Shared kernel** | Theme (both surface modes), atomic UI, i18n string tables, analytics hook stub | Emotion + tokens; folders `theme/`, `ui/`, `i18n/`, `analytics/` |
| **API module** | `baseApi`, axios instance, interceptors, `axiosBaseQuery` / `baseQueryWithAuth`, mock adapter on that instance. The only I/O boundary | TypeScript types, RTK Query, axios, `axios-mock-adapter` |

## 5. Boundaries & contract candidates

Most handoffs are **in-process imports**, not network APIs. The only contract worth writing now is the **API module’s request/response shape** — mocks today, real HTTP later. **Session 1.11 is done: the contract is `architecture/11-interface-contracts.md`** (five operations, envelope, error model, transport profiles; ADR-0016). The table below is the boundary map; doc 11 §1 is the authoritative inventory, and it adds the persisted-auth-blob and `@env` config seams this table never listed.

| Boundary | What crosses it | Sync / async | Data owner | Likely contract style | Notes for 1.11 |
|----------|-----------------|--------------|------------|----------------------|----------------|
| Shell → Auth, Storefront, Shared, API | Screen / reducer / middleware registration | Sync | Shell owns composition | None (in-process public exports) | Do not OpenAPI the shell |
| Auth → Shared · Storefront → Shared | Atoms, tokens, i18n, analytics stub | Sync | Shared kernel | None | Public component/token API only |
| Auth ↛ Storefront | Nothing | — | — | — | Shell reads auth selectors to switch navigators |
| Auth / Storefront → API module (via **RTK Query hooks** only) | **Five operations** — login, **`/me`**, **HomeFeed**, **ContinueWatching**, and **`/containers/{id}/resources`** (horizontal paging, named in 1.11) | Async (RTK Query; mock latency/failure on the axios instance) | API module owns wire shapes and `baseApi`; features own injected endpoints + auth slice | **Yes** — TS interfaces + doc 11 tables (ADR-0016) | Transport is **axios** behind `baseQueryWithAuth`, not `fetch`. Screens never import `axios`. **Shapes locked in doc 11** (OQ-22/23/26 closed). JWT on operations 2–5 via request interceptor |
| API module → future HTTP backend | Same payloads over the network | Async | Backend team later (not built here) | **Same contract** — doc 11 §6.5 records the transport profile that differs | Not a component we build; do not invent a mock server just to have HTTP |

## 6. Key flows

### Flow 1 — Sign-in (success, inline error, and restore)

1. App shell boots. **redux-persist** rehydrates the **auth slice only** from **`react-native-encrypted-storage`** (Keychain / EncryptedSharedPreferences). `/me` cache is empty until `getMe`.
2. If NetInfo reports no network, the shell shows the **no-internet overlay** (doc 15) on top of whichever navigator the session implies; it does not unmount it. Reconnect or `REINTENTAR` hides the overlay and resumes auth vs storefront from auth state.
3. If a session exists, shell shows the **cold-start loading screen**, then `getMe` + `getHomeFeed` + `getContinueWatching` (doc 04). Success → app navigator, **dark** theme, composed home. `getMe` 401 or expired mock JWT → `baseQueryWithAuth` clears session + `resetApiState()` → Welcome.
4. If not, shell applies the **light** theme and mounts the unauthenticated navigator.
5. Auth shows Welcome (Shared atoms). CTA → email → password (two-step, Phase 1a).
6. Submit: Auth calls **`useLoginMutation`** → `baseApi` → axios instance (mock adapter or HTTP). No screen imports `axios`.
7. **Success:** mock returns a JWT (7-day `exp`) → `login.fulfilled` writes the auth slice → persist writes it → request interceptor will attach the token → **`getMe` hydrates the user from cache** → same three-query gate as cold start → shell switches to the app navigator and **dark** theme.
8. **Failure:** mock fails on demand → same `ApiError` as the future API → credentials screen renders the reference inline error (red underline + message). Not an alert, not a local `if (password !== …)` branch. Not the no-internet overlay. `INVALID_CREDENTIALS` does **not** clear a session.

### Flow 2 — Storefront load and card tap

1. Shell mounts Storefront as the home tab after the boot gate (other tabs inert).
2. Storefront **composes** `[hero container] + [progress container] + [other HomeFeed containers]` from the two feed responses (ADR-0006 / ADR-0007). Pagination wrappers dispatch `loadMore` via RTK Query (`getHomeFeed` / `getContainerResources`). End-reached calls `loadMore`.
3. When the storefront screen is shown again, Storefront **silently refetches ContinueWatching** and replaces that **`progress` container** only.
4. Card tap goes through the **same handler** that will later navigate → Alert with the title (Phase 1). Tile components do not know about navigation.

## 7. Build vs. buy

Phase 1 is a mobile demo with mocks. Almost everything is built in-app; we only take OSS libraries, not hosted products.

| Capability | Build / buy | Notes |
|------------|-------------|--------|
| Auth (demo credentials, session slice) | **Build** (Phase 1 mock) | No Auth0 / Firebase / Cognito SDK. Phase 3 **buys** a managed IdP **behind our API** (ADR-0009); a real JWT is still a swap at the API module |
| Content / home feed | **Build** (fixtures + mock adapter) | No CMS, no mock HTTP server |
| Storefront UI, theme, atoms | **Build** | Emotion (`@emotion/native` + `@emotion/react` ThemeProvider) + tokens |
| HTTP + persistence + connectivity | **Buy (OSS)** | axios, **RTK Query** (`@reduxjs/toolkit`), Redux Toolkit, redux-persist, **`react-native-encrypted-storage`**, `@react-native-community/netinfo` |
| Navigation | **Buy (OSS)** | React Navigation |
| Payments, search, email, push, video CDN | **Neither** | Out of Phase 1 |
| Analytics | **Build stub** | Hook + `console.log`; no vendor |
| Backend / IdP / Bitrise | **Not now** | Backend is a future swap; CI is Phase 2 |

**Hard dependencies:** React Native, Redux Toolkit (includes **RTK Query**), React Navigation (with its required peers **`react-native-screens`** and **`react-native-safe-area-context`**), **`@emotion/native`** + **`@emotion/react`**, axios, **`axios-mock-adapter`** (Phase 1; removed in Phase 3), redux-persist, `react-native-encrypted-storage`, `@react-native-community/netinfo`, **`react-native-svg`** (renders the SVG brand and placeholder-art assets, and the icon set — ADR-0013), **`react-i18next` + `i18next`** (doc 07 §9). No cloud vendor lock-in in Phase 1. **Not a dependency:** `styled-components` (ADR-0019).

The four additions were surfaced in STEP-1.7: the assets decided in 1.2 are SVG, and the UI renders no native headers, so SVG rendering and safe-area insets are load-bearing rather than optional. See ADR-0013.

## 8. Import rules & source layout

### 8.1 Source layout

The five components of §4 map onto directories. This was implicit until 1.11 needed a stable path
for the wire types (doc 11 §3) — the layout below is the one already latent in this doc: §2 names
Shared's internals as `shared/theme/`, `shared/components/`, … and the import-rule table below has always
referred to `features/`.

```
src/app/                       shell — composition root, navigation, store, error boundary
src/features/auth/               ← reference module (STEP-6.2); copy this shape for new features
src/features/storefront/         ← migrates to same shape in STEP-6.4 (legacy `ui/` until then)
src/shared/{theme,components,i18n,analytics}/
src/shared/assets/placeholder-art/   bundled placeholder key art (STEP-3.4)
src/api/
  ├── baseApi.ts               createApi (empty endpoints); reducerPath `api`
  ├── sessionCleared.ts        `createAction('session/cleared')` — the API module's clear signal (§8.2)
  ├── types/                   wire types — the contract's authoring source (doc 11 §2)
  ├── mocks/                   axios-mock-adapter + fixtures (typed, never `any` — doc 11 §11.2)
  ├── client/                  axios instance, interceptors, axiosBaseQuery, baseQueryWithAuth
  └── integration/             T2 suites + the wired-world factory (doc 12 §2; STEP-3.5)
```

#### 8.1.1 Feature module template (mandatory)

Every feature under `src/features/<name>/` follows the **auth** layout below. **Storefront** must match after STEP-6.4. **New features use this from day one.** Do not create `features/*/ui/` or `screens/*/components/`.

| Path | Required | Purpose |
|------|----------|---------|
| `api.ts` | When feature injects RTK endpoints | `injectEndpoints` on `baseApi`; hooks exported here. |
| `components/atoms/` | If feature-only atoms exist | Single-purpose UI not shared across features. |
| `components/molecules/` | If composed feature UI exists | e.g. `CredentialsForm`, `PortraitTile`. |
| `components/organisms/` | If large feature sections exist | e.g. `WelcomeHero`, `HomeFeedList`. |
| `components/index.ts` | When `components/` is non-empty | Barrel exports. |
| `helpers/` | If pure utilities exist | Validation, type guards, mappers — **not** under `screens/` or `components/`. **Tested** helpers use a **unit subdirectory** (see below). |
| `hooks/` | If feature owns data/composition hooks | e.g. storefront pagination (omit in auth 1a). **Tested** hooks use a **unit subdirectory** (see below). |
| `screens/<ScreenName>/` | When feature owns routes | `<ScreenName>.tsx` (logic/JSX), **`<screenName>Screen.styles.ts`** (all `@emotion/native` styled components), optional `<screenName>Layout.ts`. **No `components/` subfolder.** |
| `screens/index.ts` | When feature owns routes | Navigator-facing screen exports. |
| `state/slices/<slice>/` | When feature owns Redux state | Slice + co-located unit tests. Selectors live in `state/selectors/`, not on the slice file. |
| `state/actions/` | When imperative store writers exist | e.g. `logout.ts` — dispatches slice actions + cache resets. |
| `state/selectors/` | When feature owns Redux state | One file per slice (e.g. `auth.ts`). |
| `assets/` | Optional | Feature-local static media (e.g. welcome posters). |
| `README.md` | Yes | Documents this feature's tree; keep in sync with disk. |

**Reference — `features/auth/` (actual, STEP-6.2):**

```
features/auth/
  README.md
  api.ts
  api.integration.test.ts
  sessionRestore.integration.test.ts
  assets/
    welcome/
      index.ts
      poster_*.png
      qc_wordmark.png
  components/
    atoms/
      AuthGradientBackground.tsx
      BrandStrip.tsx
    molecules/
      CredentialsForm.tsx
    organisms/
      AuthSheetLayout.tsx
      WelcomeHero.tsx
    index.ts
  helpers/
    emailValidation.ts
    isApiError.ts
  screens/
    WelcomeScreen/
      WelcomeScreen.tsx
      welcomeScreen.styles.ts
      welcomeLayout.ts
    Login/
      LoginScreen.tsx
      loginScreen.styles.ts
      loginLayout.ts
    PlaceholderScreen/
      PlaceholderScreen.tsx
      placeholderScreen.styles.ts
    index.ts
  state/
    slices/
      auth/
        authSlice.ts
        authSlice.test.ts
    actions/
      logout.ts
    selectors/
      auth.ts
```

**Target — `features/storefront/` (STEP-6.4):** same tiers; `hooks/` at feature root; **no `state/`** in Phase 1a (RTK Query cache only). See `features/storefront/README.md`.

Screens import feature UI from `../../components/{atoms,molecules,organisms}/…`. Shell imports screens from `features/<feature>/screens`. Selectors are read via `features/<feature>/state/selectors/`.

#### Tested unit subdirectories (`helpers/` and `hooks/`)

When a helper, hook, or pure utility is **non-trivial enough to warrant a unit test**, it lives in its **own subdirectory** under `helpers/` or `hooks/` — same pattern as `screens/<ScreenName>/` and `state/slices/<slice>/`:

```
helpers/placeholderArt/
  placeholderArt.ts
  placeholderArt.test.ts
  index.ts              # re-exports the public API
hooks/usePaginatedContainers/
  usePaginatedContainers.ts
  usePaginatedContainers.test.ts
  index.ts
```

| Rule | Detail |
|------|--------|
| **When** | The module has (or needs) a co-located `*.test.ts` — T1 logic, mappers, pagination, i18n formatters, etc. |
| **When not** | One-liner guards, trivial validators, or glue with no dedicated test stay as a **single flat file** (e.g. `helpers/isApiError.ts`). |
| **Entry file** | `<unitName>.ts` beside `<unitName>.test.ts` — not `index.ts` as the implementation. |
| **Barrel** | Optional `index.ts` re-exports the public surface so callers import `helpers/placeholderArt`, not the inner path. |
| **Shared types** | Cross-hook types may stay at `hooks/types.ts` when several hooks share them. |
| **Integration tests** | Stay at feature root (e.g. `api.integration.test.ts`) — not inside a unit folder. |
| **Redux slices** | Already use `state/slices/<slice>/` + co-located test; same principle. |

**Auth (Phase 1a):** `helpers/emailValidation.ts` and `helpers/isApiError.ts` remain flat — simple, untested. Add a subdirectory when a test file is introduced.

**Storefront:** all tested helpers and hooks follow this layout (post STEP-6.4).

#### Unit subdirectories (`shared/components/`)

Shared UI follows the **same subdirectory rule** as `helpers/` and `hooks/`. When a component in `shared/components/{atoms,molecules,organisms}/` has a co-located **`<ComponentName>.styles.ts`** or **`*.test.ts`**, it lives in **`shared/components/<tier>/<ComponentName>/`**:

```
shared/components/molecules/TabBarItem/
  TabBarItem.tsx
  TabBarItem.styles.ts
  index.ts
shared/components/molecules/SectionHeader/
  SectionHeader.tsx
  SectionHeader.test.ts
  index.ts
```

| Rule | Detail |
|------|--------|
| **When** | The component has (or needs) a co-located styles file or unit test. |
| **When not** | Simple atoms/icons with inline styled definitions and no test stay as a **single flat file** (e.g. `atoms/Button.tsx`, `icons/CastIcon.tsx`). |
| **Entry file** | `<ComponentName>.tsx` beside `<ComponentName>.styles.ts` / `<ComponentName>.test.ts` — not `index.ts` as the implementation. |
| **Styles file** | `<ComponentName>.styles.ts` — all `@emotion/native` `styled.*` for that component (object callback form). The `.tsx` file holds logic/JSX only. |
| **Barrel** | `index.ts` re-exports the public surface so callers import `shared/components/molecules/TabBarItem`, not the inner path. |
| **Icons** | Stay under `shared/components/icons/` as flat files unless an icon gains a styles file or test. |

Feature-local components under `features/*/components/` follow the same rule when they gain a styles file or test.

**Styling (mandatory):** no inline styles in feature screens or components — use **`@emotion/native` `styled.*` only** (no `StyleSheet.create`). **Every screen folder** includes `<screenName>Screen.styles.ts` beside `<ScreenName>.tsx`; the screen file holds logic/JSX only. Screen-local layout numbers live in `<screenName>Layout.ts`. Colors and typography come from theme tokens inside styled callbacks, not literal hex/rgba (A2). The only exception is **animated runtime values** (e.g. press opacity) on an `Animated.*` wrapper when the value cannot be static.

**`src/api/`, not `src/modules/api/`** — no `modules/` prefix appears anywhere in this
architecture, and `features/` is already the established word. The tree is created by **STEP-2.2**.
**STEP-3** transcribes the wire types from doc 11 §7 into `src/api/types/` (doc 11 §3.1).

**`src/api/integration/` is test-only** (added STEP-3.5). It holds the T2 suites and the
`*.factory.ts` that builds their world — a real `createStore()`, `baseApi.middleware`, the shared
axios instance, and the adapter, wired together (doc 12 §2, §4.2). It lives under `src/api/`
because what it tests is the API module; its import of `src/app/store/` is a **test-time** import,
which §8.2's rules do not govern (doc 12 §4.1). Nothing reachable from the app entry imports it,
and `mocks/factories.test.ts` is what keeps that true.

**`src/shared/assets/` is a bundle folder, not a code one** (added STEP-3.4). It holds the
placeholder key art the demo fixtures name — copied unchanged from
`architecture/assets/placeholder-art/`, since DF10 forbids Disney/Marvel/Star Wars/hulu/ESPN marks
and real key art in the codebase or the assets at any phase. Fixtures reference these files by
**filename string**, because `Card.artwork` is a `string` on the wire exactly as a real backend
would send a URL; resolving one of those strings to a renderable asset is the storefront card
component's job in **STEP-5**, which is what keeps a `require()` handle out of wire data and
`react-native-svg` plus the Metro SVG transformer out of STEP-3.

### 8.2 Import rules

| From → To | Allowed? |
|-----------|----------|
| Shell → Auth, Storefront, Shared, API module | **Yes** — composition root. Registers `baseApi.reducer`, `baseApi.middleware`, and `injectStore(store)` |
| Auth → Shared · Storefront → Shared | **Yes** |
| Auth / Storefront → `src/api/types/` | **Yes** — schema only |
| Auth / Storefront → own `api.ts` hooks (`useLoginMutation`, `useGetHomeFeedQuery`, …) | **Yes** — `injectEndpoints` on `baseApi` |
| Auth / Storefront → `src/api/client/` or `axios` / `fetch` | **No** |
| Auth / Storefront → `src/api/sessionCleared.ts` | **Yes** — the API module *declares* the clear signal, the auth slice *reduces* it. See below |
| API module → `features/` | **No** — including the auth slice. This is what `sessionCleared` exists to avoid |
| Auth → Storefront · Storefront → Auth | **No** (including the other feature's `api.ts`) |
| Shared → `features/` | **No** |
| Persist | **Auth slice only**, into **`react-native-encrypted-storage`** — not AsyncStorage, not the RTK Query cache |

**The `sessionCleared` seam** (added STEP-3.2). `baseQueryWithAuth` must clear the session on
`UNAUTHORIZED` (ADR-0017 / ADR-0020), but the auth slice belongs to the Auth feature and the row
above forbids the API module importing one. The API module therefore owns the *event* and the
feature owns the *reaction*: `src/api/sessionCleared.ts` exports
`createAction('session/cleared')`, `baseQueryWithAuth` dispatches it alongside
`baseApi.util.resetApiState()`, and the auth slice reduces it through `extraReducers`. The
dependency direction stays **feature → api**, and the API module is buildable and testable with no
auth slice in existence — which is how STEP-3 shipped ahead of STEP-4.

Redux: **Redux Toolkit** slices (auth only, besides `baseApi`) plus **RTK Query** so the API boundary stays visible (not hidden behind `createAsyncThunk`, not handwritten per-operation middleware).

## Decision Summary

| # | Decision | Choice | Rationale | Forecloses / tradeoff |
|---|----------|--------|-----------|-----------------------|
| 1 | Client surfaces | RN iOS + Android only | Matches docs 01/02; demo is the mobile app | No web/admin/CLI/API-as-product in Phase 1 |
| 2 | Top-level components | Five in-app: shell, auth, storefront, shared kernel, API module | No backend component — we do not build a server | A Phase-1 backend service or mock HTTP process |
| 3 | Shared kernel grain | One component; folders `theme/`, `ui/`, `i18n/`, `analytics/` | Extraction path later without package ceremony under the deadline | npm workspaces / a global types module |
| 4 | Architecture style | **Feature-based Clean Architecture** inside a modular monolith; shell is composition root | DF5 extractability + two-dev split without services; glossary Decision 2 | Microservices; features importing each other; a `domain/usecases/data` tree per feature |
| 5 | Contract candidate | API module wire shapes only — **locked in doc 11** (ADR-0016) | Only boundary that will cross a network later | Formal contracts between in-process modules |
| 14 | Source layout | `src/{app,features,shared,api}/` — **`src/api/baseApi.ts`** + feature `api.ts` injectEndpoints (§8.1) | The layout was already latent in §2 and §8.2; 1.11 needed a stable path for the wire types | A `modules/` convention; deciding folder structure at scaffold time |
| 6 | HTTP client | axios behind RTK Query `baseApi`; interceptors attach JWT and map `ApiError` (**ADR-0020**) | Single instance; Phase 1 `axios-mock-adapter` so interceptors run | `fetch`/`axios` from screens or hooks; handwritten async middleware |
| 7 | Session persistence | redux-persist, auth slice, **`react-native-encrypted-storage`** (Keychain / EncryptedSharedPreferences) | Real JWT later is a payload change, not a storage rewrite (OQ-16 closed) | AsyncStorage for tokens; persisting the RTK Query cache; custom Keychain module |
| 8 | State libraries | RTK + **RTK Query** + React Navigation | Teaching pattern stays visible; generated hooks are the I/O boundary | Zustand/MobX, Expo Router, Expo, `createAsyncThunk` for I/O |
| 9 | Build vs. buy | Build app + mocks; buy OSS libs only; no BaaS in Phase 1. Phase 3 IdP is buy-behind-API (ADR-0009). **Emotion** for theme (ADR-0019) | Phase 1 has no real backend | Auth0/Firebase/CMS as Phase-1 dependencies; `styled-components` |
| 10 | Connectivity | Shell-owned NetInfo overlay | One gate matching the reference; restore by auth state | Per-feature offline screens; last-known-home cache |
| 11 | Storefront paging | Feature hooks + paginated mocks | Organized loadMore; contract can page in Phase 3 | One-shot full-catalog payload |
| 12 | User profile | Memory-only **`getMe` cache**; JWT on every cold start | Matches production; persist stays auth-only (ADR-0003) | Persisting `userName`; a separate user slice; profile inside the auth slice |
| 13 | Home composition | Two GETs of **`Container[]`**; CW `progress` inserted under hero (ADR-0006 / ADR-0007) | Silent CW reload; real API split; one Container/Card component | Single home payload; distinct CW types |

## Open Questions

| ID | Question | Owner | Feeds into |
|----|----------|-------|------------|
| ~~OQ-16~~ | ~~Persist backend library: `react-native-encrypted-storage` vs a thin `react-native-keychain` adapter~~ **Resolved (1.3a):** `react-native-encrypted-storage` | — | closed |
| ~~OQ-17~~ | ~~Mock strategy: axios-mock-adapter on the real instance vs a separate mock client behind the same functions~~ **Resolved (1.11) then reversed (1.14 / ADR-0020):** **`axios-mock-adapter` on the real instance** so interceptors run in Phase 1 | — | closed |
| ~~OQ-18~~ | ~~Application repo name when created~~ **Resolved (planning session):** `quasar-qc-plus-mobile-app` at `Code/quasar-qc-plus-mobile-app/` (renamed STEP-6.1) | — | closed |
| ~~OQ-19~~ | ~~Pagination wire format: cursor vs offset, envelope fields~~ **Resolved (1.4):** opaque `nextCursor`. JSON names → 1.11 (OQ-22) | — | closed |

**OQ-12** is closed (planning session: Dev A = Raul Angel, Dev B = Andres Montoya). **OQ-02** is closed (doc 04). **OQ-10** is now expressed concretely as doc 11 §14's Phase-3 checklist plus **OQ-34**.

## Version Log

| Version | Date | STEP | Change |
|---------|------|------|--------|
| v0.1.0 | 2026-08-16 | STEP-1.3 | Initial draft from architecture session |
| v0.2.0 | 2026-08-16 | STEP-1.3a | Shell owns NetInfo connectivity overlay; storefront pagination hooks; persist engine locked to `react-native-encrypted-storage`. Closed OQ-16; opened OQ-19. |
| v0.3.0 | 2026-08-16 | STEP-1.4 | User slice + `/me`; two feed endpoints + boot loader; closed OQ-19. |
| v0.3.1 | 2026-08-17 | STEP-1.5 | HomeFeed/CW are `Container[]` + `resources: Card[]`; CW variant `progress` (ADR-0007). |
| v0.3.2 | 2026-08-17 | STEP-1.6a | Phase 3 IdP is buy-behind-API (ADR-0009); Phase 1 mock auth unchanged. |
| v0.3.3 | 2026-08-17 | STEP-1.7 | §7 hard dependencies completed: `react-native-svg`, `react-native-screens`, `react-native-safe-area-context`, `react-i18next`/`i18next` (ADR-0013). Design system is `architecture/07-ui-design-system.md`. |
| v0.3.4 | 2026-08-17 | STEP-1.10 | §4 shell gains the **root error boundary** (doc 10 §5.2). No new dependency — §7's hard-dependency list is unchanged. |
| v0.3.5 | 2026-08-17 | STEP-1.11 | §5 boundary table points at doc 11 and names the **fifth operation** (`/containers/{id}/resources`). §8 gains the **source-layout table** (§8.1, `src/api/`) and is renamed; import rules move to §8.2. Closed OQ-17. No dependency change. |
| v0.4.0 | 2026-08-17 | STEP-1.14 | §2 names **feature-based Clean Architecture**. I/O is **RTK Query `baseApi`** + axios interceptors; mocks are `axios-mock-adapter` (ADR-0020). Theme is **Emotion** (ADR-0019). User/content slices replaced by RTK Query cache. Reversed OQ-17. |
| v0.4.1 | 2026-08-17 | planning session | Closed OQ-18: application repo is `quasar-disney-mobile-app`. |
| v0.4.3 | 2026-08-18 | STEP-6.1 | Repo renamed to `quasar-qc-plus-mobile-app` at `Code/quasar-qc-plus-mobile-app/`; native module `QCPlusApp`, display **QC+**. |
| v0.4.3 | 2026-08-18 | STEP-3.2 | §8.1 tree gains `src/api/sessionCleared.ts`; §8.2 records the **`sessionCleared` seam** — the API module declares the clear signal, the auth slice reduces it — and states explicitly that the API module never imports a feature (PLAN Q1). |
| v0.4.4 | 2026-08-18 | STEP-3.4 | §8.1 tree gains **`src/shared/assets/placeholder-art/`** — bundled placeholder key art the demo fixtures name by filename string. Resolution of that string to a renderable asset is STEP-5's (PLAN Q4); no dependency change. |
| v0.4.5 | 2026-08-18 | STEP-3.5 | **§8.2's shell wiring landed**: `createStore()` registers `baseApi.reducer` and `baseApi.middleware`, and the composition root calls `injectStore(store)` — the two lines STEP-2 left as stubs. `api` is in the root-state type and stays off the persist whitelist (ADR-0003, DF3). `createStore()` gains an optional `extraMiddleware` slot, appended after `baseApi.middleware`, because a `configureStore` store is sealed and the 401 reaction is dispatched from *inside* the chain where a `store.dispatch` wrapper cannot see it; the T2 suite records actions through it. §8.1 tree gains **`src/api/integration/`** (test-only). No dependency change. |
| v0.4.6 | 2026-08-18 | STEP-6.2 | **§8.1.1 Feature module template** — mandatory auth-parity layout (`screens/`, `components/{atoms,molecules,organisms}/`, `helpers/`, `state/slices|actions|selectors/`). Auth migrated on disk; storefront migrates STEP-6.4; new features copy auth from day one. |
| v0.4.10 | 2026-08-19 | — | §8.1.1 **Tested unit subdirectories** — helpers/hooks with co-located `*.test.ts` live under `helpers/<name>/` or `hooks/<name>/`; simple untested utilities stay flat. Storefront migrated; auth helpers unchanged. |
| v0.4.9 | 2026-08-19 | — | §8.1.1 **mandatory `<screenName>Screen.styles.ts`** per screen folder; auth Welcome + Placeholder migrated. |
| v0.4.8 | 2026-08-19 | — | §8.1.1 **Emotion styled only** — no `StyleSheet.create` in feature modules; co-located `*Screen.styles.ts` pattern (auth `loginScreen.styles.ts`). |
| v0.4.7 | 2026-08-19 | — | §8.1.1 **no inline styles** in feature screens/components (`styled.*` only; animated runtime merge excepted). Auth tree: combined **`LoginScreen`**. |
