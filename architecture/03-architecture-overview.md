# Doc 03 — Architecture Overview & Component Boundaries

**Version:** v0.4.0
**Status:** Draft
**Last updated:** 2026-08-17 (STEP-1.14)
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

Internal partition of Shared is folders, not packages: `shared/theme/`, `shared/ui/`, `shared/i18n/`, `shared/analytics/`. There is **no types module** — wire types live in the API module, feature types live with the owning feature, component props live next to the component.

Code lives in a **single application repo**, named when the foundation STEP creates it. No backend repo is created by this project.

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
Shared's internals as `shared/theme/`, `shared/ui/`, … and the import-rule table below has always
referred to `features/`.

```
src/app/                       shell — composition root, navigation, store, error boundary
src/features/auth/
  └── api.ts                   injectEndpoints: login, getMe
src/features/storefront/
  └── api.ts                   injectEndpoints: getHomeFeed, getContinueWatching, getContainerResources
src/shared/{theme,ui,i18n,analytics}/
src/api/
  ├── baseApi.ts               createApi (empty endpoints); reducerPath `api`
  ├── types/                   wire types — the contract's authoring source (doc 11 §2)
  ├── mocks/                   axios-mock-adapter + fixtures (typed, never `any` — doc 11 §11.2)
  └── client/                  axios instance, interceptors, axiosBaseQuery, baseQueryWithAuth
```

**`src/api/`, not `src/modules/api/`** — no `modules/` prefix appears anywhere in this
architecture, and `features/` is already the established word. Created by the scaffold STEP, which
carries doc 11 §3.1's obligation to transcribe the wire types from doc 11 §7.

### 8.2 Import rules

| From → To | Allowed? |
|-----------|----------|
| Shell → Auth, Storefront, Shared, API module | **Yes** — composition root. Registers `baseApi.reducer`, `baseApi.middleware`, and `injectStore(store)` |
| Auth → Shared · Storefront → Shared | **Yes** |
| Auth / Storefront → `src/api/types/` | **Yes** — schema only |
| Auth / Storefront → own `api.ts` hooks (`useLoginMutation`, `useGetHomeFeedQuery`, …) | **Yes** — `injectEndpoints` on `baseApi` |
| Auth / Storefront → `src/api/client/` or `axios` / `fetch` | **No** |
| Auth → Storefront · Storefront → Auth | **No** (including the other feature's `api.ts`) |
| Shared → `features/` | **No** |
| Persist | **Auth slice only**, into **`react-native-encrypted-storage`** — not AsyncStorage, not the RTK Query cache |

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
| OQ-18 | Application repo name when created | Eng leadership | Planning session / foundation STEP |
| ~~OQ-19~~ | ~~Pagination wire format: cursor vs offset, envelope fields~~ **Resolved (1.4):** opaque `nextCursor`. JSON names → 1.11 (OQ-22) | — | closed |

Carried forward: OQ-12 (who is Dev A / Dev B). **OQ-02** is closed (doc 04). **OQ-10** is now expressed concretely as doc 11 §14's Phase-3 checklist plus **OQ-34**.

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
